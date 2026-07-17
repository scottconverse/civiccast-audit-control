# Owner decision — ship alongside, feature-diverge, sunset on evidence (2026-07-17)

Decided by Scott Converse (owner). Supersedes the charter's implied
"migrate then deprecate" end-state.

**The model:**

1. **Ship alongside.** The WSL product and the native Windows product are
   BOTH released and offered — one repository, shared Python core, two
   installers, both tracked by the release-truth manifest. Neither replaces
   the other by decree.
2. **Feature-diverge.** The WSL line enters maintenance mode: bugfixes only,
   owned by the rc-line coder. All new features — including the owner's
   next-version feature suite — land in the native line only. Migration is
   pulled by value, not pushed by deprecation.
3. **Sunset on evidence.** The WSL line retires when adoption and field data
   say stations no longer need it — an owner call made on evidence, never a
   scheduled cutoff. Until then it remains fully supported.

**Standing implications:**

- The dual-runtime exclusion guard (charter WS4) remains mandatory before
  any machine can have both products installed.
- The migration machinery (WS2 restore + cutover inventory) becomes the
  voluntary upgrade path rather than a one-time cutover event; it must stay
  releasable as an operator-facing feature.
- Program scope discipline is unchanged: WSL-line defects are observed and
  recorded, never fixed by this program; the restore machinery remains the
  single dual-use exception.
- CHARTER.md sections 1 (end state), 7 (migration framing), and 8 (final
  gate rows) are amended accordingly (see the accompanying charter change,
  cross-reviewed per governance rules).
