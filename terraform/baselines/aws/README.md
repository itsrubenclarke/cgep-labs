# Lab 5.2: AWS Security Services Baseline

Region: `us-east-1`
Trail: `cgep-lab-mgmt`
Trail bucket: `cgep-lab-cloudtrail-46549452`
Evidence: `evidence/lab-5-2/security-hub-findings.json` (15 findings, 1 CRITICAL, 14 LOW)

## CloudTrail (AU-2, AU-12, AU-10)

`aws_cloudtrail.mgmt` is a multi-region trail with `include_global_service_events = true`, so it captures management events (AU-2, event logging; AU-12, audit generation) across every region and IAM/global services from one place, not just `us-east-1`.

`enable_log_file_validation = true` is the AU-10 control. It makes CloudTrail write an hourly digest file signed by an AWS-managed key, so tampering with a delivered log after the fact is detectable rather than just theoretically possible.

**Artifact:** `terraform output -raw trail_name` returns `cgep-lab-mgmt`, and `aws cloudtrail get-trail-status --name cgep-lab-mgmt` confirmed `IsLogging: true` with a `LatestDeliveryTime` after apply, so the trail is live and delivering, not just declared.

## Security Hub (RA-5, SI-4)

`aws_securityhub_account.this` enables the hub, and two standards subscriptions were applied on top of it: NIST 800-53 Rev 5 and AWS Foundational Security Best Practices. Security Hub pulls findings from its own native checks plus anything else feeding it (GuardDuty, Config, Inspector) and normalizes them into one list, which is the RA-5 (vulnerability scanning) and SI-4 (system monitoring) story: one place an assessor can query instead of five consoles.

**Artifact:** `aws securityhub describe-hub` returned the hub ARN for this account's `us-east-1` hub, and `evidence/lab-5-2/security-hub-findings.json` holds 15 findings captured about an hour after apply, well within the lab's 30 minute expectation.

## Config (CM-2, CM-6, CM-8)

Not deployed. Config wasn't blocked by an SCP in this account, it was left out to avoid the ongoing per-recorder cost while the account is only being used for this lab. The gap is not silent: Security Hub's own CRITICAL finding, "AWS Config should be enabled and use the service-linked role for resource recording" (`security-control/Config.1`), is sitting at the top of the findings file. That finding is itself the CM-2/CM-6/CM-8 evidence: the account is reporting its own configuration-recording gap in machine-readable form.

## Finding: `enable_default_standards` drift on the account resource

Security Hub was enabled through the console before Terraform ever ran against it (the API-based `EnableSecurityHub` call was rejected with `SubscriptionRequiredException` until a one-time console enablement was done). That console flow left `enable_default_standards` at its default of `true`, so the first `terraform apply` against the imported account showed `aws_securityhub_account.this` as `must be replaced`. Applying it re-created the account resource and, along with it, auto-subscribed the account to every default standard AWS ships, not just the two declared in `security_hub.tf`.

`aws securityhub get-enabled-standards` afterward listed eight subscriptions: the two Terraform manages (NIST 800-53, FSBP) plus six more it never declared (CIS Foundations v1.2.0 and v5.0.0, PCI-DSS, NIST 800-171, the AI security best practices standard, and the resource tagging standard). Each of those bills per check once its findings finish populating, which matters here since the whole point of Chapter 5's cost note is keeping the bill to under a few dollars.

Fixed by disabling the six extra standards in the console, then adding `enable_default_standards = false` to `security_hub.tf` and re-applying. Since that argument can only be set at account creation, the fix required one more destroy-and-recreate of `aws_securityhub_account.this`. `aws securityhub get-enabled-standards` now shows exactly the two intended standards, matching what the Terraform config declares.

## Cleanup

Evidence was captured into `evidence/lab-5-2/security-hub-findings.json` before teardown. `terraform destroy` removes the trail, the trail bucket (via `force_destroy = true`), and detaches all standards subscriptions including the six undeclared ones above.
