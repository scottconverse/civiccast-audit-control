# WS1 Release-Truth Round-4 Re-audit Verdict

- **Verdict:** `CHANGES_REQUIRED`
- **Slice:** `ws1-release-truth`
- **Requested gate:** Charter section 8 row 1, scoped to manifest, read-only checker, and external observer
- **CivicCast repository:** `https://github.com/scottconverse/civiccast`
- **Audited SHA:** `fafdee3bed292618cbccd6291efd3605081fcd0f`
- **Merge base:** `44463a197f326b7e607c31c56f28047ff1b00fb0`
- **Prior audited SHA:** `bcd43a8868c1ce71840dbcc4a42ac365dec6c092`
- **Branch / PR:** `claude/ws1-release-truth` / CivicCast PR #287, targeting `program/native-windows`
- **Codex thread ID:** `019f6cfb-27e1-7240-82ba-b81aaca03ca3`
- **Audit date:** 2026-07-16 (America/Denver)
- **Auditor environment:** Windows 10.0.26200 x64; PowerShell; Python 3.14.5; Git 2.54.0.windows.1; `uv` 0.11.15
- **Execution posture:** `sandbox=danger-full-access`; `approval_policy=never` (non-interactive)
- **Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-ws1-release-truth-fafdee3b`
- **Worktree state:** detached at the audited SHA and clean after verification
- **Activity relay:** one round-specific updatable comment on CivicCast PR #287: <https://github.com/scottconverse/civiccast/pull/287#issuecomment-4997710872>
- **Heartbeat result:** required start and phase-transition milestones were relayed; no relay failure affected the audit

## Scope and gate boundary

The re-audit covered the complete `bcd43a88..fafdee3b` fix diff, both changed evidence files, final release-truth code/tests/manifest/README state, live PR #287, the current audit-control observer pin, audit-control issue #3, current GitHub releases, and the exact live `main` README at `61e0449ecd5565889a711da5c6f9e1e1eb51af58`.

The sole round-3 code finding, CC-WS1-004, is closed. The one live-main drift remains honest-red external state rather than a tooling defect. Generated README tooling, assets/hashes/signing truth, release-event triggers, and full charter-row closure remain outside this scoped verdict.

## Severity rollup

- Blocker: 0
- Critical: 0
- Major: 1
- Minor: 0
- Nit: 0

## Finding

### CC-WS1-005 - Major - PR #287 still presents the superseded round-3 SHA as current

- **Surface:** live CivicCast PR #287 description, specifically its Audit history, Evidence, and Observer paragraphs.
- **Evidence:** GitHub reports the live PR head as `fafdee3bed292618cbccd6291efd3605081fcd0f`, and audit-control commit `eea633b95ab36f9a001c409f77f217169967c540` pins the observer to that same SHA. The PR description instead calls `bcd43a88` "current, re-audit requested," says the four evidence blobs were verified at `bcd43a88`, and says the observer pin is `bcd43a88`.
- **Consequence:** the public handoff and audit surface binds its current evidence and observer claims to a superseded commit. This regresses the documentation reconciliation that closed CC-WS1-005 in round 3 and makes the PR body disagree with both the source under audit and the live governance observer.
- **Acceptance criterion:** update PR #287's description so its audit history identifies `fafdee3b` as the current round-4 audit SHA, its evidence statement binds the four blob checks to `fafdee3b`, and its observer statement identifies the live `fafdee3b` pin (audit-control `eea633b`). Preserve the honest-red issue #3 and read-only/no-release-authority statements.

## Closed round-3 finding

- **CC-WS1-004:** closed. `run_evidence.py` now executes `git hash-object --path <rel> <rel>`. For the LF-committed checker, an asserted CRLF working copy produced a different raw hash but the path-filtered hash exactly equaled the committed blob; a substantive tamper produced a different path-filtered hash. All four final log blob lines equal `git rev-parse fafdee3b:<path>`.

## Regression status of the earlier closed set

- **CC-WS1-001 / CC-WS1-002 / CC-WS1-003 / CC-WS1-006:** no code regression found. The 22-test suite and three-file lint command pass, including successor integrity, malformed-input fail-closed behavior, authenticated live lookup behavior, and evidence-runner lint coverage.
- **CC-WS1-005:** regressed only on the live PR description as detailed above; the manifest and repository documentation remain consistent with the implemented behavior.

## Commands and proofs re-run

- Exact repository, live PR head/base, detached SHA, origin, and clean worktree checks -> all bound to `fafdee3bed292618cbccd6291efd3605081fcd0f`; PR remains open against `program/native-windows`.
- `git diff --check bcd43a88..fafdee3b` and scoped/final-state review -> no whitespace error; the source diff is one path-aware hash invocation plus regenerated evidence text.
- `python -m pytest tests/policy/test_release_truth.py -q` -> `22 passed in 3.66s`, exit 0.
- `uv run ruff check scripts/policy/check_release_truth.py tests/policy/test_release_truth.py .agent-runs/native-windows/ws1-release-truth/evidence/run_evidence.py` -> all checks passed, exit 0.
- Four evidence identities -> every logged blob ID equals both `git hash-object --path <path> <path>` and `git rev-parse HEAD:<path>` for checker, manifest, tests, and README.
- CRLF filter falsification -> committed checker blob `9c9cbb7ed334e394ff676eb833058c98e01923b8`; raw CRLF hash `8fb7d6985cf87092301129d5267632dd52893e3e`; path-filtered CRLF hash `9c9cbb7ed334e394ff676eb833058c98e01923b8`; path-filtered tamper hash `63822b1d659b147541bdcf04be03713b91357e94`.
- Authenticated checker against audited checkout README -> `release-truth: PASS`, exit 0.
- Authenticated checker against exact live-main README at `61e0449e` -> exactly `DRIFT: README.md never mentions the current release v1.0.0-rc14`, exit 1.
- Audit-control observer workflow at `eea633b` -> exact `fafdee3b` checker pin; permissions remain `contents: read` and `issues: write`; no CivicCast write or release authority exists in the path.
- Hosted observer history -> authenticated evaluation remains proven at run 29542897314; issue #3 remains open with exactly the known rc14/rc15 drift and owner-visible context. A duplicate hosted run was not dispatched because this round changes only the evidence runner, not the checker or manifest.
- Live PR description reconciliation -> failed: three current-state statements still name `bcd43a88`.

## Audit-lite dimensions

- **Correctness and security:** the filter-aware fix and release checker pass; the observer remains read-only and fail-visible.
- **UX:** no product UI changed. Public audit-state UX is inaccurate because the PR description names the prior SHA as current.
- **Documentation:** repository evidence documentation matches the code; the live PR description does not match the live head or observer pin.
- **Tests:** 22 committed tests plus the independent CRLF normalization/tamper control pass.
- **Runtime:** authenticated developer-machine live checks pass for the audited README and report exactly one expected drift for current `main`.
- **Escalation:** no audit-team escalation is warranted; the remaining finding is confined to the public PR audit surface.

## Confidence classes and boundaries

- **Class 1:** exact detached SHA, complete round-4 diff/final state, live PR metadata/body, governance pin, issue state, and observer authority trace.
- **Class 2:** 22-test suite, three-file lint, four blob comparisons, and independent CRLF/tamper falsification.
- **Class 3:** authenticated developer-machine live GitHub check against both audited and exact live-main README bytes.
- **External observer:** the current exact pin is verified statically; the authenticated hosted execution path remains proven by run 29542897314.
- **Honest-red external state:** `main` omits rc14 while the manifest holds rc15 as staging. This is one correctly detected drift, not release authority.
- No merge, tag, release, cutover, CivicCast branch edit, implementation fix, or PR-description edit was performed.

## Final rationale

`CHANGES_REQUIRED` is bound only to `fafdee3bed292618cbccd6291efd3605081fcd0f`. The filter-aware evidence fix meets the round-3 acceptance criterion, and the checker, tests, lint, live behavior, and observer pin remain sound. Advancement is nevertheless unsupported while the live PR's authoritative handoff surface identifies the superseded commit as current and attributes the current evidence and pin to it.

This verdict does not merge or advance the slice. Scott Converse remains the owner and tie-breaker; an eventual exact-SHA `AUDIT_PASS` would authorize merge only to `program/native-windows`, never CivicCast `main`, and never a tag or release.
