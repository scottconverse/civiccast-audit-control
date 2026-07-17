# Owner decision — severity calibration (2026-07-17)

Decided by Scott Converse (owner, tie-breaker), resolving the open governance
item recorded in the round-10 WS1 verdict:

> "Real defects only. Wording errors in documents we can give near release.
> 40% of audits, almost a million tokens, is far too high a price to pay for
> spelling and wording errors. So, batch the nits."

**Rule, effective immediately:**

1. Only functional defects — wrong behavior, unsound proofs, false claims
   that change what a reader would DO — block an audit round.
2. Prose/wording precision findings (imprecise counts, stale phrasing,
   missing path names, and similar) are recorded as Minor, batched, and may
   be closed in any later round up to the relevant near-release gate. They
   never individually trigger a re-audit round.
3. False claims of fact remain functional defects (they corrupt the record);
   imprecise or incomplete-but-not-misleading wording is a nit.

Coder and auditor both apply this rule; disagreement about whether a given
finding is "functional" goes to Scott as always.
