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
| build-image | `build-image.yml` | Builds and pushes an image to `ghcr.io/attom-ai/forgen`; push requires this repo's owner to be granted GHCR access to that package |
| approve | `approve.yml` | Pauses on a GitHub Environment's required reviewers |
| deploy | `deploy.yml` | Rolls the approved image out to the target environment (k3s today) — applies its manifests so the running service picks up the new build |

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
`type: choice`) — it's validated inside `build-image.yml`, which fails
fast with `::error::` on anything other than the three values above.

## Usage

```bash
gh workflow run Pipeline -R <owner>/attogen-deployment-util \
  -f repo=Attom-AI/attogen -f branch=QA -f service=pipeline -f environment=k3s
```
