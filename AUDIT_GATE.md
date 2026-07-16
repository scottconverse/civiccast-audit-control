# CivicCast Native-Windows Audit Gate

This gate controls advancement through `CHARTER.md`. It is mandatory for every native-Windows slice.

## Gate rule

A slice advances only when all of the following are true:

1. The coder submits an audit request naming one exact CivicCast commit SHA, slice ID, charter gate, claims, evidence paths, exact commands, negative controls, known gaps, and requested execution posture.
2. The auditor checks out that SHA in a separate detached worktree and records the resolved repository, SHA, dirty state, sandbox, and approval policy.
3. The auditor reads the complete scoped diff and final state, re-runs the named falsifications and material positive proofs, and distinguishes static, worktree-runtime, service-session, hardware, and clean-machine confidence.
4. The auditor emits the activity milestones required by `AUDIT_PROTOCOL.md` so Scott can see that the audit is active and which proof phase is running.
5. The auditor writes exactly one canonical verdict under `verdicts/`, bound to the CivicCast SHA and Codex thread ID.
6. Scott approves any owner gate. Neither coder nor auditor merges, tags, releases, or cuts over on its own verdict.

## Verdicts

Allowed verdicts:

- `AUDIT_PASS`: every acceptance criterion in the requested gate is proved for the audited SHA; open gaps are outside that gate and stated.
- `CHANGES_REQUIRED`: one or more in-scope defects or proof gaps must be corrected.
- `BLOCKED`: the auditor cannot complete required verification because a named external dependency or owner decision is unavailable.

A verdict applies only to the exact source SHA it names. It is never inherited by a descendant, amended commit, rebased branch, merge commit, or tag. The audit-control commit containing the verdict is provenance for the record, not a substitute for the audited source SHA.

## Proof boundaries

- A clean detached worktree proves source isolation, not a clean machine.
- A developer box, restored VM snapshot, container, or preprovisioned runner is not a pristine Windows installation.
- Element presence is not device operation; a successful short run is not a soak; a user-session run is not session-0 proof.
- Signing or attestation proves provenance and integrity, not that a manual observation was truthful.
- Skipped, unavailable, or unexecuted required checks prevent `AUDIT_PASS` for the gate they support.

## Authority and trust boundary

Security-sensitive findings use the private embargo lane in `AUDIT_PROTOCOL.md`. No public finding detail or canonical verdict is committed until Scott authorizes disclosure.

This public repository is a process and history boundary, not a credential boundary. Claude, Codex, and Scott may operate through the same GitHub credential or PAT; GitHub committer identity therefore does not authenticate which agent produced a request or verdict. Trust comes from exact-SHA binding, Codex thread IDs, rerunnable evidence, review history, and Scott's merge/tag/cutover authority.

Coder/auditor disagreements go to Scott Converse, the owner and tie-breaker. The transport and shared PAT grant no release authority.

See `AUDIT_PROTOCOL.md` for request, execution, security, and record formats.
