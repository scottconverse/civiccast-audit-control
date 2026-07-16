# Canonical Verdicts

Store verdicts as:

`verdicts/<slice>/<40-character-civiccast-source-sha>.md`

Every record follows `AUDIT_PROTOCOL.md` section 7 and binds one verdict to one CivicCast source SHA and one Codex thread ID. Verdicts never carry across commits.

Do not commit secrets or exploitable vulnerability details. Use the security-embargo lane in `AUDIT_PROTOCOL.md`.

Bootstrap Phase 0 begins with this directory empty. Scott approves the initial audit-control seed and CivicCast PR #283 before normal verdict issuance begins.
