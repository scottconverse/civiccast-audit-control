# Authority records (PROPOSED - requires auditor + owner ratification)

**Status: PROPOSED by the coder 2026-07-17 (specs design-review round 8
identified that the claims-evidence rule's authority-record format needs an
authoritative definition HERE, not only in a product-repo spec). Takes
effect for claims verification only when (a) the auditor ratifies the format
in a design-review round or verdict, and (b) the owner accepts the claims
trust root (see the owner-acceptance register in the civiccast specs
README). Verdict records under `verdicts/` are UNAFFECTED - this extends
the family, it does not modify AUDIT_PROTOCOL section 8.**

An authority record certifies that a specific NON-CI evidence record proves
a specific claim control at a specific product commit. It is the structured,
signature-bound edge between the product repo's claims registry and this
repository's evidence store, defined so a correctly-signed review can never
be resolved against the wrong or later evidence bytes.

## Canonical path

`authority/<claim-id>/<control-id>/<source-sha-40hex>.md`

One record per (claim, control, source SHA). Records are CREATE-ONLY: a
change to an existing path is a violation; supersession is a NEW source SHA.

## Required structured fields (same field syntax as verdict records)

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
2. All three path segments equal their body fields exactly.
3. The referenced evidence blob is byte-identical to what the verifier
   fetches at the referenced evidence commit (binding to bytes, never to a
   path).
4. Exactly one record per path; duplicates or post-creation modifications
   invalidate.
5. An authority record never grants a gate - gates remain verdicts under
   `verdicts/` per AUDIT_PROTOCOL section 8. Authority records certify
   evidence for CLAIMS only.
