# Deploying to Cloud Run (free test environment) — one-time setup

This is a one-time, manual setup you run yourself with `gcloud` (nobody but
the project owner can do this — it grants IAM permissions in your GCP
project). Once it's done, `.github/workflows/ci-cd.yml`'s `deploy-cloud-run`
job deploys automatically on every push to `main`, with no long-lived
secrets stored in GitHub: authentication uses **Workload Identity
Federation** (WIF) — GitHub's own OIDC token is exchanged for short-lived
GCP credentials at deploy time, so there is no service-account key to leak
or rotate.

**Project used:** `ai-deployment-509116`
**Region:** `us-central1`
**Cloud Run service name:** `hello-world-api`
**Image source:** the same `ghcr.io/yeapkl/ai-assisted-api` image the
`docker-publish` job already builds and pushes — Cloud Run pulls it
directly, so nothing is duplicated into Artifact Registry.

## 0. Prerequisites

- `gcloud` CLI installed and authenticated as a project owner/editor:
  `gcloud auth login`
- Billing enabled on the project (Cloud Run's free tier — 2 million
  requests/month, 360k GB-seconds — applies automatically, but Cloud Run
  still requires a billing account on file even if you never exceed it).
- The GHCR package must be **public** so Cloud Run can pull it without
  extra registry credentials: on GitHub, go to the package page
  (`https://github.com/yeapkl/ai-assisted-api/pkgs/container/ai-assisted-api`)
  → **Package settings** → **Change visibility** → **Public**. (One-time,
  not a `gcloud` step.)

## 1. Run this once

```bash
export PROJECT_ID="ai-deployment-509116"
export REGION="us-central1"
export REPO="yeapkl/ai-assisted-api"          # GitHub owner/repo
export DEPLOYER_SA="github-actions-deployer"
export RUNTIME_SA="cloud-run-runtime"
export POOL_ID="github-pool"
export PROVIDER_ID="github-provider"

gcloud config set project "$PROJECT_ID"

# --- Enable the APIs this setup needs -------------------------------------
gcloud services enable \
  run.googleapis.com \
  iamcredentials.googleapis.com \
  secretmanager.googleapis.com \
  sts.googleapis.com \
  cloudresourcemanager.googleapis.com

# --- Two service accounts: one that deploys, one the service runs as -----
# Separating these means the GitHub Actions identity can deploy new
# revisions but never gets standing access to the secrets the *running*
# service reads.
gcloud iam service-accounts create "$DEPLOYER_SA" \
  --display-name="GitHub Actions deployer (CI/CD)"

gcloud iam service-accounts create "$RUNTIME_SA" \
  --display-name="Cloud Run runtime identity (hello-world-api)"

# Deployer can deploy Cloud Run services...
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${DEPLOYER_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.admin"

# ...and can hand off ("actAs") the runtime SA to new revisions.
gcloud iam service-accounts add-iam-policy-binding \
  "${RUNTIME_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --member="serviceAccount:${DEPLOYER_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# --- Workload Identity Federation: trust GitHub's OIDC tokens -------------
gcloud iam workload-identity-pools create "$POOL_ID" \
  --location="global" \
  --display-name="GitHub Actions pool"

gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_ID" \
  --display-name="GitHub Actions provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository=='${REPO}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Only THIS GitHub repo (not the whole org/account) may impersonate the
# deployer SA — the --attribute-condition above scopes token acceptance,
# this scopes which identity a valid token from that repo can become.
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

gcloud iam service-accounts add-iam-policy-binding \
  "${DEPLOYER_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/attribute.repository/${REPO}"

# --- JWT_SECRET_KEY lives in Secret Manager, never in the workflow file ---
openssl rand -hex 32 | gcloud secrets create jwt-secret-key --data-file=-

gcloud secrets add-iam-policy-binding jwt-secret-key \
  --member="serviceAccount:${RUNTIME_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# --- Print what you need for step 2 ---------------------------------------
echo "GCP_WORKLOAD_IDENTITY_PROVIDER=projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/providers/${PROVIDER_ID}"
echo "GCP_SERVICE_ACCOUNT=${DEPLOYER_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
echo "GCP_RUNTIME_SERVICE_ACCOUNT=${RUNTIME_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
```

## 2. Add repository variables on GitHub

None of the three values above are secret (they're identifiers, not
credentials — WIF is what makes that true), so they go under
**Settings → Secrets and variables → Actions → Variables** (not Secrets):

| Variable | Value |
|---|---|
| `GCP_PROJECT_ID` | `ai-deployment-509116` |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | the `projects/.../providers/github-provider` string the script printed |
| `GCP_SERVICE_ACCOUNT` | the `github-actions-deployer@...` email the script printed |
| `GCP_RUNTIME_SERVICE_ACCOUNT` | the `cloud-run-runtime@...` email the script printed |

## 3. Deploy

Push to `main` (or merge a PR into it). The `deploy-cloud-run` job runs
after `docker-publish` and:

1. Authenticates via WIF (no key file, ever).
2. Deploys the just-published `ghcr.io/yeapkl/ai-assisted-api:sha-<commit>`
   image to the `hello-world-api` Cloud Run service, running as the
   `cloud-run-runtime` service account.
3. Mounts `jwt-secret-key` from Secret Manager as the `JWT_SECRET_KEY` env
   var — the value is never visible in workflow logs or GitHub secrets.
4. Curls `/health` on the deployed URL as a smoke test.

The service URL is printed in the job's output (and in the GitHub
"Environments" tab, since the job declares a deployment environment).

## Rotating the JWT secret later

```bash
openssl rand -hex 32 | gcloud secrets versions add jwt-secret-key --data-file=-
```

Cloud Run reads `:latest` on each new revision, so the next deploy (or a
manual `gcloud run services update hello-world-api --region=us-central1`
with no other changes) picks it up. Rotating it invalidates every
previously issued access/refresh token, by design.

## Tearing this down

```bash
gcloud run services delete hello-world-api --region=us-central1
gcloud secrets delete jwt-secret-key
gcloud iam workload-identity-pools providers delete github-provider \
  --location=global --workload-identity-pool=github-pool
gcloud iam workload-identity-pools delete github-pool --location=global
gcloud iam service-accounts delete github-actions-deployer@ai-deployment-509116.iam.gserviceaccount.com
gcloud iam service-accounts delete cloud-run-runtime@ai-deployment-509116.iam.gserviceaccount.com
```
