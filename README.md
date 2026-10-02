# rf_ecr_sync

GitHub Actions workflows that mirror a curated list of [RapidFort](https://www.rapidfort.com/) hardened images from `quay.io/rfcurated` into a private container registry.

| Workflow | Destination | Auth |
|----------|-------------|------|
| [`rf_sync.yaml`](rf_sync.yaml) | Amazon ECR | Runner's existing AWS credentials (`aws ecr get-login-password`) |
| [`rf_acr_sync.yaml`](rf_acr_sync.yaml) | Azure Container Registry | OIDC via `azure/login` → `az acr login --expose-token` |

Both workflows share the same logic and the same image catalogue; only the destination registry and its login step differ.

## How it works

1. **Load catalogue** — reads the `RF_IMAGES_TO_SYNC` JSON (or a `workflow_dispatch` override) and validates it.
2. **Sync (matrix)** — one job per catalogue image, up to 4 in parallel:
   - installs [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane)
   - logs crane into the destination registry
   - exchanges RapidFort credentials for a short-lived quay.io token via `docker-credential-rfcurated` (refreshed every 10 tags)
   - for each listed tag:
     - resolves the newest timestamped snapshot (`<tag>-YYYYMMDDTHHMMSS`) if one exists, otherwise uses the tag as-is
     - copies all platforms to `<repo>:<image>-<tag>-<YYYYMMDD>` only if that digest isn't already in the registry
     - moves `<repo>:<image>-<tag>-latest` to the source digest (self-healing, no blob re-upload)
   - writes a per-image table to the job summary

Only tags listed explicitly in the catalogue are synced. There is no "sync everything" mode, by design.

### Triggers

- Weekly, Mondays at 12:00 UTC
- Manually from the Actions tab, with an optional `images` JSON override

## Catalogue format

```json
[
  {
    "image": "rfal",
    "ecr_repo": "rfcurated",
    "acr_repo": "rfcurated",
    "tags": ["al23-rfcurated", "al23-fips-rfcurated"]
  }
]
```

| Field | Used by | Description |
|-------|---------|-------------|
| `image` | both | Source image under `quay.io/rfcurated` (e.g. `rfal`, `envoyproxy/envoy`) |
| `ecr_repo` | ECR (and ACR fallback) | Destination repository in ECR |
| `acr_repo` | ACR | Destination repository in ACR. If omitted, `ecr_repo` is used, so one catalogue can drive both workflows |
| `tags` | both | Exact quay.io tags to sync. Must not be empty |

Images that share a destination repo are told apart by an image-name prefix on the tag (`/` becomes `-`), e.g. `rfcurated:envoyproxy-envoy-<tag>-latest`.

## Setup

### Shared

| Kind | Name | Notes |
|------|------|-------|
| Environment variable | `RF_IMAGES_TO_SYNC` | Defined on a GitHub **Environment** named `RF_IMAGES_TO_SYNC` (Settings → Environments), not at repo level |
| Secret | `RF_ACCESS_ID` | RapidFort access ID |
| Secret | `RF_SECRET_ACCESS_KEY` | RapidFort secret access key |

### ECR (`rf_sync.yaml`)

- Set `ECR_REGISTRY` and `AWS_REGION` in the workflow's `env:` block.
- The runner must already have AWS credentials with push access to the target ECR repo. ECR repos must exist before the first push.

### ACR (`rf_acr_sync.yaml`)

- Set `ACR_NAME` and `ACR_REGISTRY` (`<name>.azurecr.io`) in the workflow's `env:` block.
- Create an Entra ID app registration or user-assigned managed identity and:
  - grant it **AcrPush** on the registry
  - add a federated credential for this repo. The sync job doesn't use a GitHub Environment, so the subject is branch-based, e.g. `repo:Gravitas-Security/rf_ecr_sync:ref:refs/heads/main`
- Add these repo secrets:

| Secret | Value |
|--------|-------|
| `AZURE_CLIENT_ID` | Client ID of the app registration / identity |
| `AZURE_TENANT_ID` | Entra ID tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Subscription containing the registry |

ACR creates repositories on first push, so nothing needs to be created ahead of time.
