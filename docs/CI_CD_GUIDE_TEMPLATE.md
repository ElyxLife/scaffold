# Universal CI/CD Guide for Poly-Component Projects on Google Cloud Platform

## Overview
This template provides a **project-agnostic**, secure, and fully automated CI/CD pipeline reference that you can apply to any repository containing multiple deployable units (frontend, backend, infrastructure, serverless functions, etc.).  The design eliminates long-lived credentials by relying exclusively on **Workload Identity Federation (WIF)** and enforces **modular deployments** so that only the parts of a system that change are rebuilt and released.

> **How to use this document**  
> 1. Copy the file into your repository (e.g. `docs/CI_CD_GUIDE.md`).  
> 2. Perform a **global search-and-replace** on the placeholders wrapped in `<<...>>`.  
> 3. Follow the phase-by-phase instructions, committing changes as you progress.

---

## Placeholder Reference
| Placeholder           | Description                                                         |
| --------------------- | ------------------------------------------------------------------- |
| `<<ORG_NAME>>`        | GitHub organisation or username that owns the repository            |
| `<<REPO_NAME>>`       | Name of **this** repository                                         |
| `<<PROJECT_ID_ENV>>`  | GCP project ID for a specific environment (`staging`, `prod`, etc.) |
| `<<PROJECT_NUM_ENV>>` | GCP project **number** for the same environment                     |
| `<<ENV>>`             | The current environment (`staging`, `prod`, etc.)                   |
| `<<COMPONENT>>`       | One of `frontend`, `backend`, `infra`, `cloud-functions`, etc.      |

You can keep placeholder tokens in committed YAML/Markdown files—CI jobs will dynamically substitute them using Bash/`envsubst` if desired.

---

## Repository Structure
```
.
├── .github/workflows/           # GitHub Actions workflows
├── frontend/                    # React / Next.js / etc.
├── backend/                     # FastAPI / Django / etc.
├── infra/                       # Terraform root module & scripts
│   ├── modules/                 # Re-usable Terraform modules
│   └── scripts/                 # Deployment helpers (shell)
├── components/                  # Optional shared libs / packages
└── docs/                        # Project documentation (incl. this guide)
```
Feel free to rename directories; just update paths in scripts and workflow files accordingly.

---

## Core Tenets
1. **Zero GitHub Secrets** – All authentication to Google Cloud uses WIF; _no_ JSON keys are stored in GitHub.  
2. **Auto-Generated Secrets** – Terraform creates runtime secrets and stores them in Secret Manager.  
3. **Modular Deployments** – Each component builds & deploys independently; a failure in one does not block others.  
4. **Environment Isolation** – Separate GCP projects (or at minimum, distinct workloads) per environment.  
5. **Pipeline-as-Code** – Every action (including WIF setup) is automated via CLI or Terraform.

---

## Implementation Phases
### Phase 1 — Modular Deployment Scripts
Create shell scripts in `infra/scripts/` (or a directory of your choice):

| Script                      | Purpose                                                                   |
| --------------------------- | ------------------------------------------------------------------------- |
| `load-secrets.sh`           | Source `*.tfvars.secrets.<ENV>` files and export as environment variables |
| `deploy-backend.sh`         | Build Docker image & deploy backend to Cloud Run / App Engine             |
| `deploy-frontend.sh`        | Build static assets & upload to Cloud Storage / Firebase Hosting          |
| `deploy-cloud-functions.sh` | Deploy serverless functions                                               |
| `deploy-infrastructure.sh`  | Run `terraform apply` for shared resources                                |

> **Tip**: Each script should accept an `ENV` argument (`staging`, `prod`, etc.) so that the same logic works across environments.

#### Script Conventions & Best Practices
• **Argument order**: `./deploy-<component>.sh <ENV> [version-tag] [project-id]` – keeps calls consistent across automation.  
• **Exit on error**: Start each script with `set -euo pipefail` for reliability.  
• **Docker Buildx cache** (backend):
```bash
docker buildx build \
  --cache-from=type=registry,ref="$IMAGE_URI:cache" \
  --cache-to=type=registry,ref="$IMAGE_URI:cache",mode=max \
  --tag "$IMAGE_URI:$VERSION" .
```
• **Terraform targeting**: Scripts that deploy only one module call `terraform apply -target=module.<name>`; full orchestrator scripts run plain `terraform apply`.
• **Idempotency**: Scripts should be **safe to re-run**; make them detect existing Docker tags and skip builds when unchanged.

### Phase 2 — Workload Identity Federation
1. Enable required APIs:
   ```bash
   gcloud services enable iamcredentials.googleapis.com sts.googleapis.com \
       cloudresourcemanager.googleapis.com --project "${PROJECT_ID_ENV}"
   ```
2. Create a Workload Identity Pool and OIDC provider (one per environment):
   ```bash
   gcloud iam workload-identity-pools create "github-actions-${ENV}" \
       --project="${PROJECT_ID_ENV}" --location="global" \
       --display-name="GitHub Actions Pool – ${ENV}"

   gcloud iam workload-identity-pools providers create-oidc "github" \
       --project="${PROJECT_ID_ENV}" --location="global" \
       --workload-identity-pool="github-actions-${ENV}" \
       --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
       --attribute-condition="assertion.repository=='<<ORG_NAME>>/<<REPO_NAME>>'" \
       --issuer-uri="https://token.actions.githubusercontent.com"
   ```
3. Create a service account `github-actions-${ENV}@${PROJECT_ID_ENV}.iam.gserviceaccount.com` and grant the roles it needs (Cloud Run admin, etc.).
4. Allow the OIDC provider to impersonate the service account:
   ```bash
   PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID_ENV}" --format='value(projectNumber)')
   gcloud iam service-accounts add-iam-policy-binding \
     "github-actions-${ENV}@${PROJECT_ID_ENV}.iam.gserviceaccount.com" \
     --role="roles/iam.workloadIdentityUser" \
     --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-actions-${ENV}/attribute.repository/<<ORG_NAME>>/<<REPO_NAME>>"
   ```
All of the above can be automated via `scripts/setup-workload-identity.sh`.

### Phase 3 — GitHub Actions Workflows
Split into **CI** (validation) and **CD** (deployment) workflows.  Example matrix:

```yaml
name: CI – Validate
on:
  pull_request:
    paths:
      - backend/**
      - frontend/**
      - infra/**
      - components/**

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
      infra: ${{ steps.filter.outputs.infra }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend: {"-" : "backend/**"}
            frontend: {"-" : "frontend/**"}
            infra: {"-" : "infra/**"}

  backend-tests:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install deps
        run: uv sync  # ✨ project uses uv for Python deps
      - name: Run tests
        run: uv run pytest
```

Add corresponding **deploy-<COMPONENT>** jobs in a separate workflow triggered on `push` to `main` (auto-deploy to `staging`) and manual dispatch for `prod`.

The deployment job skeleton:
```yaml
jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    permissions:
      contents: read
      id-token: write  # required for OIDC
    steps:
      - uses: actions/checkout@v4
      - id: project
        run: echo "project_id=$(grep '^project_id' infra/terraform.tfvars.${{ inputs.env }} | cut -d '"' -f2)" >> $GITHUB_OUTPUT
      - uses: google-github-actions/auth@v2
        with:
          project_id: ${{ steps.project.outputs.project_id }}
          workload_identity_provider: projects/<<PROJECT_NUM_ENV>>/locations/global/workloadIdentityPools/github-actions-${{ inputs.env }}/providers/github
          service_account: github-actions-${{ inputs.env }}@${{ steps.project.outputs.project_id }}.iam.gserviceaccount.com
      - name: Run deployment script
        env:
          ENV: ${{ inputs.env }}
        run: infra/scripts/deploy-backend.sh "$ENV"
```

Repeat for other components, swapping scripts and build steps as needed.

#### Change Detection & Component-Scoped Workflows
Using GitHub’s `paths:` filter (or `dorny/paths-filter` action) keeps CI fast and inexpensive by running jobs **only** when their code changes. Extend the filter to include additional paths such as `cloud-functions/**` or `mobile/**`.

#### Additional Recommended CI Checks
| Component       | Extra Validation                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------ |
| Backend         | `uv run mypy backend` for type-checking, **Docker build test** (`docker build` with `--load` but no push)    |
| Frontend        | `npm test -- --coverage` to enforce unit-test thresholds, production `npm run build` to catch compile errors |
| Infra           | `terraform validate` and `terraform plan -lock=false -detailed-exitcode` to detect drift                     |
| Cloud Functions | Python syntax validation or function-specific test suite                                                     |

> **Deployment order tip**: If the frontend consumes API URLs produced by Terraform outputs, deploy **backend first**, then frontend so the latest endpoint is available at build time.

> **Terraform targeting tip**: When deploying a single component, speed things up with `terraform apply -target=module.<name>`; reserve full plans for infra-wide changes.

---

## Secret Management Strategy
1. **Public Configuration** – `terraform.tfvars.<ENV>` (committed) contain *non-secret* values like `project_id` and `project_number`.  
2. **Local Secrets** – `terraform.auto.tfvars.secrets.<ENV>` (un-tracked) store initial secrets required during `terraform apply`.  
3. **Runtime Secrets** – Google Secret Manager manages credentials used by applications.  
4. **Generated Secrets** – Terraform `random_password` and `random_string` resources.

---

## Security Best Practices
* Short-lived OIDC tokens (15 min) instead of JSON keys.  
* Principle of Least Privilege for service accounts and IAM roles.  
* Branch protection & required status checks before merging to `main`.  
* Manual approval gates for production.  
* Quarterly permission reviews & dependency updates.

---

## Code Quality Automation
Modern pipelines should fail fast when **formatting, linting, or test coverage** rules are violated. Configure language-specific tools and run them both locally (pre-commit) and in CI.

### Recommended Tooling
| Stack     | Formatter / Linter | Example CI Command                   |
| --------- | ------------------ | ------------------------------------ |
| Python    | Ruff               | `uv run ruff format --check backend` |
| JS / TS   | Prettier + ESLint  | `npm run lint --workspaces`          |
| Terraform | `terraform fmt`    | `terraform fmt -check -recursive`    |

### GitHub Actions Snippet
```yaml
- name: Verify Code Quality
  runs-on: ubuntu-latest
  strategy:
    matrix:
      component: [backend, frontend, infra]
  steps:
    - uses: actions/checkout@v4

    # Node toolchain for frontend linting
    - name: Set up Node
      if: matrix.component == 'frontend'
      uses: actions/setup-node@v4
      with:
        node-version: 20

    - name: Install JS deps
      if: matrix.component == 'frontend'
      run: npm ci --prefer-offline --no-audit

    - name: Run ESLint/Prettier
      if: matrix.component == 'frontend'
      run: npm run lint

    - name: Install Python deps
      if: matrix.component == 'backend'
      run: uv sync

    - name: Check Ruff formatting
      if: matrix.component == 'backend'
      run: uv run ruff format --check ./backend

    - name: Terraform format check
      if: matrix.component == 'infra'
      run: terraform fmt -check -recursive
```

### Local Pre-commit Hook
Install once:

```bash
uv add pre-commit
pre-commit install
```

Sample `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.5
    hooks:
      - id: ruff-format
      - id: ruff-lint
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.2.5
    hooks:
      - id: prettier
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.77.0
    hooks:
      - id: terraform_fmt
```

CI will only stay green when code is **properly formatted and lint-clean** across all components.

---

## Maintenance Checklist
- [ ] No GitHub repository secrets present.  
- [ ] Successful WIF authentication in CI and CD logs.  
- [ ] Modular deployments only rebuild changed components.  
- [ ] Production approval gates functioning.  
- [ ] Cost and performance metrics monitored.

---

## Troubleshooting
1. **OIDC Auth Failures** – Ensure `project_number` and repository filters match.  
2. **Permission Denied** – Verify `roles/iam.workloadIdentityUser` binding on the service account.  
3. **Deployment Rollback** – Check container build logs and Cloud Run revision health.

---

## Conclusion
By following this template your repository gains a fully automated, zero-key CI/CD pipeline that can be reused across multiple projects with minimal effort—just substitute the placeholders and commit the necessary workflow and script files.
