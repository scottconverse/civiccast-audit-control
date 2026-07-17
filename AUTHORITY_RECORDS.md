# Authority records v1 (AUDITOR-RATIFIED; OWNER ACCEPTANCE PENDING)

**Status: proposed by the coder at audit-control commit `082dcffa5f5defd812d6e3dfff53ae3e0fbf0c6b`
and amended + ratified by the auditor in specs design-review Round 9. The
format remains NON-AUTHORIZING until Scott explicitly accepts both (a) this
authority-record v1 contract and (b) the product trust-root pin that binds
the repository, signer roles, and the exact governance bytes below.
Acceptance of the signer key set alone is not acceptance of this record
class. Verdict records under `verdicts/` are UNAFFECTED - this extends the
family, it does not modify `AUDIT_PROTOCOL.md` section 8.**

An authority record certifies that a specific NON-CI evidence record proves
a specific claim control at a specific product commit. It is the structured,
signature-bound edge between the product repo's claims registry and this
repository's evidence store, defined so a correctly-signed review can never
be resolved against the wrong or later evidence bytes.

## Format identity and trust binding

- Format identifier: `authority-record-v1`.
- The product trust root must pin the canonical audit-control repository,
  the audit-control commit containing the owner-accepted definition, the
  Git blob ID of this file at that commit, the `keys/allowed_signers` blob,
  and the role assigned to every accepted signer.
- The verifier fetches this file from the pinned commit and requires its
  blob ID to match the pin before parsing any authority record. Drift on
  audit-control `main` never silently changes the accepted format.
- External-evidence claims remain cannot-check/non-authorizing until the
  owner has accepted both this format pin and the signer/role pin.

## Canonical path

`authority/<claim-id>/<control-id>/<source-sha-40hex>.md`

One record per (claim, control, source SHA). Records are CREATE-ONLY: a
change to an existing path is a violation; supersession is a NEW source SHA.

`claim-id` and `control-id` are lowercase path-safe slugs matching
`[a-z0-9][a-z0-9._-]{0,127}`. A slash, backslash, percent escape, `..`,
uppercase spelling, empty value, or any other spelling is malformed. Source
SHA is exactly 40 lowercase hexadecimal characters. These rules make path
identity stable across GitHub and case-insensitive Windows checkouts.

## Required structured fields (same field syntax as verdict records)

- **Authority format:** `authority-record-v1`
- **Claim:** `<claim-id>` (must equal the path segment)
- **Control:** `<control-id>` (must equal the path segment)
- **Source SHA:** `<40-hex product-repo commit>` (must equal the path segment)
- **Evidence commit:** `<40-hex audit-control commit containing the evidence record>`
- **Evidence blob:** `<40-hex git blob ID of the exact evidence record bytes>`
- **Assessment:** free text - what was checked, how, confidence boundary.

## Validity rules (enforced by the product repo's claims verifier)

1. The commit introducing the record is signature-verified against the
   PINNED `keys/allowed_signers`, with role = auditor or owner. A
   coder-key-signed authority record is invalid.
2. The authority format field is exactly `authority-record-v1` and the
   verifier has already validated the pinned governance blob above.
3. All three path segments obey the canonical grammar and equal their body
   fields exactly.
4. The evidence record is resolved at
   `evidence/<source-sha>/<claim-id>/<control-id>.json` in the referenced
   evidence commit. That canonical path must resolve to `Evidence blob`,
   and the fetched bytes must hash to the same blob ID. A matching blob at
   another path is not sufficient.
5. The authority path is added exactly once. Any duplicate logical tuple,
   alternate/non-canonical path, deletion, or post-creation modification
   invalidates the record.
6. An authority record never grants a gate - gates remain verdicts under
   `verdicts/` per AUDIT_PROTOCOL section 8. Authority records certify
   evidence for CLAIMS only.
