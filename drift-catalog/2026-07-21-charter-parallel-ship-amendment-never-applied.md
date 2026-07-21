# Charter never amended for the parallel-ship decision

- **Detected:** 2026-07-21 (America/Denver), while scoping the WS5 documentation package
- **Affected governance record:** `scottconverse/civiccast-audit-control` -> `CHARTER.md`, last modified at `ae94479` (2026-07-16 16:19 -0600)
- **Affected decision:** `decisions/2026-07-17-parallel-ship.md`
- **Severity / gate:** Major governance drift; affects program end-state framing at every gate, and gate row 7 directly
- **Owner:** Scott Converse (owner-controlled repository); amendment authored by Claude (coder) on owner instruction 2026-07-21
- **Disposition:** corrected in place; original text preserved in Git history

## Observed versus expected

The owner's parallel-ship decision (2026-07-17) states in its closing line that
"CHARTER.md sections 1 (end state), 7 (migration framing), and 8 (final gate
rows) are amended accordingly (see the accompanying charter change,
cross-reviewed per governance rules)."

**No such charter change was ever committed.** `CHARTER.md` has exactly two
commits, `1a604b7` and `ae94479`, both dated 2026-07-16 - the day *before* the
decision. Nothing has touched the file since.

The result is that the authoritative charter continued to describe the
superseded migrate-then-deprecate end state, while `STATUS.md` row 7 and
`reviews/2026-07-17-specs-design-review.md` both correctly cite parallel-ship.
The charter was the only governance document still telling the old story, and
it is the document agents are pointed at first.

**Observed consequence, not hypothetical:** on 2026-07-21 the coder asserted in
session that the native line "is intended to merge back into `main` when the
migration completes - that merge is the entire point of the program," and
reasoned about documentation scope on that basis. The owner corrected it. The
assertion was a faithful reading of the unamended charter.

## Reproduction

```bash
git -C civiccast-audit-control log --format="%h %ci %s" -- CHARTER.md
# -> only ae94479 (2026-07-16) and 1a604b7 (2026-07-16)

git -C civiccast-audit-control log --since=2026-07-17 -- CHARTER.md
# -> empty

grep -Ei "alongside|parallel|sunset|maintenance mode|feature-diverge" CHARTER.md
# -> no matches (before this correction)
```

## Closing evidence

`CHARTER.md` now carries three dated amendment notes, each citing
`decisions/2026-07-17-parallel-ship.md`:

- **Section 1** - end state: both products released and offered, one repository,
  shared Python core, two installers; WSL in maintenance mode; sunset on
  evidence, never a scheduled cutoff. Adds the governing rule that where the
  charter's original text reads as migrate-then-deprecate, the decision wins.
  Records that section 6 (dual-runtime exclusion) is unchanged and still
  mandatory.
- **Section 7** - migration framing: the WSL line is not retired at native
  equivalence; the migration machinery is the voluntary upgrade path and must
  stay releasable as an operator-facing feature.
- **Section 8** - gate row 7 retitled "LPM voluntary upgrade + rollback
  rehearsal", with a note that completing it advances one station and does not
  retire the WSL product.

## Standing lesson

A decision record that *claims* it amended another document is not evidence that
the amendment happened. Governance changes asserted in prose need the same
commit-bound verification as implementation claims - which is exactly the
principle the claims-evidence rule (WS3) exists to enforce for code. This drift
sat undetected for four days inside the repository whose purpose is detecting
exactly this.
