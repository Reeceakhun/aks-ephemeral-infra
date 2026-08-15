# aks-ephemeral-infra

Provisions short-lived AKS clusters on demand, and guarantees they're torn
down automatically — regardless of what asked for them or whether that
caller is still around.

## Why this is a separate repo

Application repos (like [`ci-cd-flask-k8s-pipeline`](https://github.com/Reeceakhun/ci-cd-flask-k8s-pipeline))
shouldn't need to know how to provision or tear down cloud infrastructure —
that's a platform concern, not an application concern. This repo owns the
full lifecycle of ephemeral Azure environments; any number of app repos can
request one without duplicating provisioning logic or teardown safety nets
across every pipeline that needs a test environment.

## How it works

```
App repo pipeline finishes build
        │
        ▼
fires repository_dispatch → this repo
        │
        ▼
provision.yml creates a tagged resource group + AKS cluster,
deploys the given image, prints the service endpoint
        │
        ▼
reaper.yml (running independently, every 5 min) checks the
age of every ephemeral-tagged resource group and deletes any
past its TTL — currently 20 minutes
```

The provisioner and the reaper are deliberately decoupled: the provisioner's
only responsibility is "create and tag correctly." The reaper's only
responsibility is "delete anything expired." Neither depends on the other
succeeding, staying alive, or even having run in the same session — so a
crashed provisioning job, a cancelled workflow, or a caller that never checks
back in all still get cleaned up on schedule.

## Tagging convention

Every ephemeral resource group is tagged with:
| Tag | Meaning |
|---|---|
| `purpose=ephemeral-demo` | marks it as something the reaper should manage |
| `created-at=<ISO8601 UTC timestamp>` | when it was created |
| `ttl-minutes=<n>` | how long it's allowed to live |

The reaper reads only these tags — there's no external database tracking
environment state. The resource group is the single source of truth for its
own lifecycle.

## Authentication

Uses OIDC federation between this repo's GitHub Actions and an Azure AD App
Registration — no client secret or long-lived credential is stored anywhere.
See [`docs/azure-setup.md`](docs/azure-setup.md) for the full setup.

## Troubleshooting: OIDC federated credential subject mismatches

Getting the federated credential's `subject` right took three attempts, and
each failure was informative enough to keep documented rather than squash away:

1. **Wrong branch in the subject** — first registered against
   `ref:refs/heads/master`, but this repo's default branch is `main`. Azure
   AD rejected the token with `AADSTS700213: No matching federated identity
   record found`.
2. **Wrong subject format entirely** — even after fixing the branch, the
   same error persisted. The error message itself revealed the actual
   subject GitHub was presenting:
   `repo:Reeceakhun@84012952/aks-ephemeral-infra@1334270331:ref:refs/heads/main`
   — newer GitHub OIDC tokens include internal numeric org/repo IDs
   alongside their names, not just plain `owner/repo`. Azure AD requires an
   exact string match, so the credential had to be recreated using that
   literal subject.
3. **Lesson:** don't hand-construct the subject from documentation examples
   — when `azure/login` fails with `AADSTS700213`, the error message
   contains the exact string Azure AD received. Copy it directly.

See [`docs/azure-setup.md`](docs/azure-setup.md) for the corrected setup steps.

## Triggering a cluster manually

Actions tab → **Provision Ephemeral AKS Environment** → Run workflow →
supply an `image_tag` (e.g. `reeceakhun/ci-cd-flask-k8s-pipeline:abc1234`).

## Triggering from another repo

The calling repo needs a token with permission to dispatch events to this
repo, then fires:

```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/Reeceakhun/aks-ephemeral-infra/dispatches \
  -d '{"event_type":"provision-env","client_payload":{"image_tag":"reeceakhun/ci-cd-flask-k8s-pipeline:abc1234"}}'
```

## Cost notes

AKS control plane is free on the Free tier. You're billed only for the node
VM(s) — a single `Standard_B2s` node for ~20 minutes costs a fraction of a
cent. The reaper's 5-minute polling interval means worst-case lifetime is
~24 minutes rather than exactly 20, a deliberate tradeoff against running
the reaper more frequently for a demo project.

## What I'd change for production use

- Scope the service principal's role assignment to something narrower than
  subscription-level Contributor (a custom role limited to resource-group
  and AKS operations)
- Add a dead-letter/alert if the reaper itself fails partway through a run,
  so an expired environment doesn't silently outlive its TTL unnoticed
- Move the TTL and polling interval to repo variables instead of hardcoded
  workflow values, so they're configurable without editing YAML