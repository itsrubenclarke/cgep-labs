# Lab 4.4: Chain of Custody — Property to Artifact Mapping

Run verified: `32378734780` (commit `38e3e7604489030d89b7c779a6a45bfa0071399c`)
Vault: `cgep-lab-grc-evidence-vault-b32df8c6`
Bundle: `runs/32378734780/evidence-32378734780-38e3e7604489030d89b7c779a6a45bfa0071399c.tar.gz`

## Authenticity

The evidence bundle carries a Cosign keyless signature (`evidence-...tar.gz.sig.bundle`), issued via Fulcio against a short-lived certificate bound to the GitHub Actions OIDC token for this workflow run. `cosign verify-blob` confirms the signing identity came from GitHub's OIDC issuer (`https://token.actions.githubusercontent.com`) for this run, rather than trusting an unverifiable claim of origin.

**Artifact:** `evidence-32378734780-....tar.gz.sig.bundle`, checked by `scripts/verify-evidence.sh`'s `cosign verify-blob` step, which printed `Verified OK`.

## Integrity

A `.sha256` sidecar was generated at signing time and is checked against the bundle on every verification run. `verify-evidence.sh` recomputes the hash and fails closed on any mismatch.

**Artifact:** `evidence-32378734780-....tar.gz.sha256`, containing `56cff4cbac10c238d6ac335204a71071cdb7460abbcb2e26df3040a92ee8f18a` — matches `receipt.json`'s recorded `sha256` and the hash recomputed directly from the vault copy.

## Timeliness

Cosign's signature is bound to a Rekor transparency log entry created at signing time, giving an independently-witnessed timestamp for when the bundle was produced — not a self-reported date in the bundle contents. `cosign verify-blob` fails if no corresponding Rekor entry exists for the signature.

**Artifact:** the Rekor entry referenced inside `evidence-32378734780-....tar.gz.sig.bundle`, confirmed present as part of the `Verified OK` result during verification.

## Preservation

The bundle is stored in an S3 bucket with Object Lock enabled (GOVERNANCE mode, default 1-day retention applied automatically on upload). The original object version cannot be deleted or altered while under retention.

**Artifact:** `receipt.json`'s `version_id` (`3wyjHG3lGmPybAGJeR.IV.ZQSYgc.ata`), confirmed via `aws s3api get-object-retention` to carry `Mode: GOVERNANCE` with `RetainUntilDate: 2026-08-21T14:13:04Z`.

## Tamper test finding

Ran the Step 5 tamper test: downloaded the bundle, appended a byte, and re-uploaded to the same key. Two things came out of it, one expected and one worth flagging:

- **Expected:** the tampered copy's hash (recomputed locally) no longer matched the `.sha256` sidecar, and `cosign verify-blob` against the tampered bytes failed signature verification — the original bytes are what was signed, so any change breaks the check.
- **Not what the lab guide implies:** the re-upload to the same key *succeeded*, creating a new S3 object version rather than being rejected outright. Object Lock's GOVERNANCE retention protects a specific **version** from deletion or replacement — it does not stop a new `PutObject` from creating a new current version at the same key. For a short window, `IsLatest` pointed at the tampered version rather than the original.
- The original locked version (`3wyjHG3lGmPybAGJeR.IV.ZQSYgc.ata`) was never touched or at risk — fetching it explicitly by version ID and re-hashing it confirmed it was byte-for-byte the same as what the receipt recorded.
- **Practical takeaway:** any consumer of this vault must verify against the `version_id` pinned in `receipt.json`, not just the bare key, since "latest at this key" is not itself a tamper-proof pointer. The tampered version was deletable only via `s3:BypassGovernanceRetention` — a permission worth restricting tightly in a real deployment, since it's the actual control standing between "tamper is detectable" and "tamper can be quietly hidden by removing the evidence that flags it."
