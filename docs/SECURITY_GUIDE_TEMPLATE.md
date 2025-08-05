# Universal Security Hardening Guide for Google-Cloud-Based Projects

This template distills modern, **key-less** security patterns and hardening practices that can be applied to **any** Google Cloud project. Replace the `<<PLACEHOLDER>>` tokens with values from your own environment and delete any sections that do not apply to your stack.

> **How to use**  
> 1  Copy this file to your repo (e.g. `docs/SECURITY.md`).  
> 2  Search-and-replace the placeholders.  
> 3  Delete optional sections you don’t need, then commit.

---

## Placeholder Reference
| Placeholder       | Description                                                     |
| ----------------- | --------------------------------------------------------------- |
| `<<PROJECT_ID>>`  | The Google Cloud project ID                                     |
| `<<PROJECT_NUM>>` | The Google Cloud **project number**                             |
| `<<ENV>>`         | Environment label (`staging`, `prod`, etc.)                     |
| `<<ORG_NAME>>`    | GitHub organisation or username                                 |
| `<<REPO_NAME>>`   | Name of your GitHub repository                                  |
| `<<MEDIA_SA>>`    | Service account that can sign URLs (e.g. `media-bucket-access`) |

---

## 1  Service-Account Key Elimination

### 1.1 Overview
Long-lived JSON keys are a security liability and are **no longer required** for the majority of Google Cloud workflows thanks to **Workload Identity Federation** (for CI/CD) and **IAM Credentials API** (for runtime signing).

### 1.2 How It Works
1. Runtime service (Cloud Run / Cloud Functions) authenticates using **its own** default service account.  
2. When a signed URL or impersonation is needed it calls **IAM Credentials API** to impersonate `<<MEDIA_SA>>@<<PROJECT_ID>>.iam.gserviceaccount.com`.  
3. IAM signs the request and returns the token—**no private key ever leaves Google’s infrastructure**.

### 1.3 Terraform Implementation
```hcl
# Allow Cloud Run to impersonate the media service account
resource "google_service_account_iam_member" "cloudrun_can_impersonate_media_sa" {
  service_account_id = google_service_account.media_service_account.name
  role               = "roles/iam.serviceAccountTokenCreator"
  member             = "serviceAccount:${data.google_project.current.number}-compute@developer.gserviceaccount.com"
}
```

### 1.4 Example Backend Code (Python)
```python
# OLD: Using private key from Secret Manager (❌)
credentials = service_account.Credentials.from_service_account_info(json.loads(service_account_key))

# NEW: Key-less via ADC (✅)
from google.auth import default
credentials, _ = default()  # Uses Cloud Run’s auto-injected token

signed_url = blob.generate_signed_url(
    version="v4",
    expiration=expiration_time,
    method="GET",
    service_account_email="<<MEDIA_SA>>@<<PROJECT_ID>>.iam.gserviceaccount.com",
)
```

---

## 2  Secrets Management Strategy
1. **Public Config** — Non-secret values (`project_id`, region) live in committed `terraform.tfvars` files.  
2. **Bootstrap Secrets** — Initial secrets for Terraform (`terraform.auto.tfvars.secrets.*`) **never** committed.  
3. **Runtime Secrets** — Stored in **Secret Manager** and accessed via IAM roles.  
4. **Generated Secrets** — Use `random_password` resources; Terraform rotates them as needed.

---

## 3  Authentication & Authorization
* Short-lived OIDC tokens (15 min) via **Workload Identity Federation** for all CI/CD interactions.  
* **Least-Privilege IAM**: start with the minimum set of roles; audit quarterly.  
* **Repository-Scoped Trust**: Workload Identity Provider condition `assertion.repository == '<<ORG_NAME>>/<<REPO_NAME>>'`.  
* Automated **token expiration & rotation**—no manual key rotation.

---

## 4  Infrastructure Hardening
| Area            | Recommended Setting / Practice                                     |
| --------------- | ------------------------------------------------------------------ |
| Projects & Env  | One GCP project per environment or strict IAM separation           |
| Network         | Private Google access, no public IPs on databases                  |
| State Storage   | Terraform state in GCS bucket with **object-versioning & locking** |
| Tags & Labels   | Consistent `env`, `owner`, and `cost-center` labels for all assets |
| Build Artifacts | Google Artifact Registry with vulnerability scanning enabled       |

---

## 5  Development-Workflow Security
* **Branch Protection**: PR reviews + status checks required on `main`.  
* **Automated Testing & Linting**: CI must pass unit tests, type checks, and linters before merge.  
* **Deployment Gates**: Manual approval required for `prod` workflows.  
* **SBOM Generation**: Enable Docker SBOM export or `cyclonedx` for language packages.  
* **Dependabot / Renovate**: Keep dependencies patched.

---

## 6  Monitoring & Auditing
| Category                | Tool / GCP Feature                                      |
| ----------------------- | ------------------------------------------------------- |
| Authentication Events   | Cloud Audit Logs → Log-based metrics & alerts           |
| Secret Access           | Secret Manager audit logs                               |
| Vulnerability Scans     | Artifact Registry CVE scanning & Cloud Security Scanner |
| IAM Anomalies           | Cloud IAM Recommender & Policy Troubleshooter           |
| Cost & Quota Monitoring | Cloud Billing Budgets & Cloud Monitoring dashboards     |

---

## 7  Regular Maintenance Checklist
- [ ] **Quarterly** IAM role review & removal of unused accounts.  
- [ ] **Monthly** dependency updates merged.  
- [ ] **Daily** monitoring of vulnerability scan results.  
- [ ] **After each release** verify no new long-lived keys exist.

---

## 8  Incident Response Quick-Start
1. **Revoke** suspicious service accounts via `gcloud iam service-accounts disable`.  
2. **Rotate** secrets in Secret Manager (set new versions, disable old).  
3. **Audit** Cloud Audit Logs for the time window of the incident.  
4. **Patch** vulnerabilities; redeploy services.  
5. **Post-mortem**: document root cause and update this guide if needed.

---

## 9  Resources
• Workload Identity Federation docs — <https://cloud.google.com/iam/docs/workload-identity-federation>  
• IAM Credentials API — <https://cloud.google.com/iam/docs/creating-short-lived-service-account-credentials>  
• Google Cloud Security Foundations Guide — <https://cloud.google.com/architecture/security-foundations>  
• Google Cloud Best Practices Center — <https://cloud.google.com/best-practices>

---

By adopting these patterns you achieve a **zero-key** posture, fine-grained access control, and a security workflow that scales with your engineering velocity.