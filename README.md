# attogen-deployment-util

A standalone deployment orchestrator for attogen's services. Dispatched
with a target repo, branch, service, and environment, it fetches that
source, runs its test suite, builds and pushes a container image, pauses
for manual approval, and deploys — one linear pipeline, each stage a
separate reusable workflow.

It lives in its own repo on purpose: it only ever reads the target repo,
through a narrowly scoped token, and never writes back to it. Nothing here
touches attogen's own history, branches, or CI.

Scope today is attogen's `converter`, `pipeline`, and `minio` services,
deployed to k3s. Inputs are plain strings rather than a fixed enum, so
other repos, services, or environments aren't structurally ruled out — but
this is the only combination actually built and exercised so far.

## Pipeline

```mermaid
flowchart LR
    A["Pipeline<br/>workflow_dispatch"] --> B["init<br/>validate"]
    B --> C["build-image<br/>build + push"]
    C --> D["approve<br/>manual gate"]
    D --> E["deploy<br/>rollout"]
```

| Stage | File | Description |
|---|---|---|
| Pipeline | `pipeline.yml` | Orchestrator — the only `workflow_dispatch` entry point |
| init | `init.yml` | Fetches the target repo and runs its test suite |
| build-image | `build-image.yml` | Builds and pushes `ghcr.io/attom-ai/forgen:util-<sha12>-<service>` (never attogen's `k3s-…` tags) and outputs its digest ref; `minio` builds nothing |
| approve | `approve.yml` | Pauses on a GitHub Environment's required reviewers |
| deploy | `deploy.yml` | On a self-hosted runner inside the cluster network: `kubectl set image` on `forgen-<service>` to the built digest, waits for the rollout, checks `/healthz`, and rolls back on failure. Never creates resources or touches ConfigMaps/Secrets; `minio` is verify-only (StatefulSet ready) |

Each stage after `Pipeline` is a reusable workflow (`on: workflow_call`) —
none of them are dispatched directly.

## Inputs

| Input | Type | Required | Default | Example (attogen) |
|---|---|---|---|---|
| `repo` | string | yes | `Attom-AI/attogen` | `Attom-AI/attogen` |
| `branch` | string | yes | `main` | `QA` |
| `service` | string | yes | — | `converter`, `pipeline`, or `minio` |
| `environment` | string | yes | `k3s` | `k3s` |

`service` isn't constrained by the input schema itself (a plain string, not
`type: choice`) — it's validated inside `build-image.yml` and again in
`deploy.yml`, which fail fast with `::error::` on anything other than the
three values above. `deploy.yml` likewise rejects any `environment` other
than `k3s`.

## Setup

One-time, out of band — none of this is done by the workflows:

| What | Why |
|---|---|
| `ATTOGEN_READ_TOKEN` secret | Fine-grained PAT, contents read-only on the target repo; used by `init` and `build-image` checkout |
| `GHCR_TOKEN` secret | Classic PAT with `write:packages` from an account with write on the `attom-ai/forgen` package (SSO-authorized if the org requires it). `GITHUB_TOKEN` can't push there from a repo outside the org |
| `approval-gate` environment | Required reviewers; `approve.yml` pauses on it |
| `k3s` environment | Optional vars `K3S_CONTEXT` (default `odcp`) and `K3S_NAMESPACE` (default `forgen`) |
| Self-hosted runner, label `k3s-util` | The cluster API is private. A runner registered to another repo can't take this repo's jobs, so register a second runner instance on the cluster host for this repo, as the same OS user as that host's kubeconfig |

Registering the runner:

```bash
gh api -X POST repos/<owner>/attogen-deployment-util/actions/runners/registration-token --jq .token
# then on the cluster host, in a fresh directory with the actions-runner tarball unpacked:
./config.sh --url https://github.com/<owner>/attogen-deployment-util --token <TOKEN> \
  --name <host>-util --labels k3s-util --unattended
sudo ./svc.sh install && sudo ./svc.sh start
```

Don't run this pipeline's deploy at the same time as attogen's own
`deploy-k3s.yml` — the two concurrency groups live in different repos and
can't see each other.

## Usage

```bash
gh workflow run Pipeline -R <owner>/attogen-deployment-util \
  -f repo=Attom-AI/attogen -f branch=QA -f service=pipeline -f environment=k3s
```
