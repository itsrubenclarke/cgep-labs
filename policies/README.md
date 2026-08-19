# Compliance Policies

Policy-as-code library mapping NIST 800-53 controls to Terraform-planned resources across GCP and AWS. Each policy is evaluated against `terraform show -json` plan output using Conftest / OPA.

## Control Coverage Matrix

| Control | Title | Framework | Severity | GCP Package | AWS Package | Resource(s) Tested |
|---------|-------|-----------|----------|-------------|-------------|--------------------|
| **AC-3** | Access Enforcement | nist-800-53 | Critical | `compliance.ac3` | `compliance.ac3_aws` | GCS buckets + firewall rules / S3 public access block |
| **CM-6** | Configuration Settings | nist-800-53 | Medium | `compliance.cm6` | `compliance.cm6_aws` | Required labels / required tags |
| **SC-28** | Encryption at Rest | nist-800-53 | High | `compliance.sc28` | `compliance.sc28_aws` | GCS CMEK / S3 server-side encryption |

---

## AC-3 Access Enforcement

**Severity:** Critical

### GCP `policies/ac3_no_public.rego`
Package: `compliance.ac3`

GCS buckets must enforce `uniform_bucket_level_access` **and** `public_access_prevention = "enforced"`. Firewall rules must not allow `0.0.0.0/0` on management ports (22, 3389).

> **Remediation:** Set `uniform_bucket_level_access = true` and `public_access_prevention = "enforced"`. For firewalls, narrow `source_ranges` or remove the rule.

### AWS `policies/ac3_no_public_aws.rego`
Package: `compliance.ac3_aws`

Every `aws_s3_bucket` must have an `aws_s3_bucket_public_access_block` referencing it, with all four flags set to `true`.

---

## CM-6 Configuration Settings

**Severity:** Medium

### GCP `policies/cm6_required_labels.rego`
Package: `compliance.cm6`

Every taggable resource must carry the four required labels: `project`, `environment`, `managed_by`, `compliance_scope`.

> **Remediation:** Add the four required labels (`project`, `environment`, `managed_by`, `compliance_scope`) to the resource.

### AWS `policies/cm6_required_tags_aws.rego`
Package: `compliance.cm6_aws`

Every taggable resource must carry the required tags.

---

## SC-28 Encryption at Rest

**Severity:** High

### GCP `policies/sc28_encryption.rego`
Package: `compliance.sc28`

Every `google_storage_bucket` must encrypt at rest with a customer-managed encryption key (CMEK).

> **Remediation:** Add an `encryption { default_kms_key_name = ... }` block referencing a `google_kms_crypto_key` you control.

### AWS `policies/sc28_encryption_aws.rego`
Package: `compliance.sc28_aws`

Every `aws_s3_bucket` must have an `aws_s3_bucket_server_side_encryption_configuration` that references it.

> **Remediation:** Add `aws_s3_bucket_server_side_encryption_configuration { bucket = aws_s3_bucket.<name>.id ... }` for the bucket.