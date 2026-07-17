# WS1 Release-Truth Round-5 Re-audit Verdict

- **Verdict:** `CHANGES_REQUIRED`
- **Slice:** `ws1-release-truth`
- **Requested gate:** Charter section 8 row 1, scoped to manifest, read-only checker, evidence, and external observer
- **CivicCast repository:** `https://github.com/scottconverse/civiccast`
- **Audited SHA:** `fafdee3bed292618cbccd6291efd3605081fcd0f`
- **Merge base:** `44463a197f326b7e607c31c56f28047ff1b00fb0`
- **Branch / PR:** `claude/ws1-release-truth` / CivicCast PR #287, targeting `program/native-windows`
- **Codex thread ID:** `019f6cfb-27e1-7240-82ba-b81aaca03ca3`
- **Audit date:** 2026-07-16 (America/Denver)
- **Auditor environment:** Microsoft Windows NT 10.0.26200.0 x64; PowerShell; Python 3.14.5; Git 2.54.0.windows.1; `uv` 0.11.15
- **Execution posture:** `sandbox=danger-full-access`; `approval_policy=never` (non-interactive)
- **Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-ws1-release-truth-fafdee3b-r5`
- **Worktree state:** fresh, detached at the audited SHA, and clean before and after verification
- **Activity relay:** <https://github.com/scottconverse/civiccast/pull/287#issuecomment-4997757515>
- **Heartbeat result:** start and every applicable phase transition were relayed; no command was expected to exceed two minutes and no relay failure occurred

## Same-path supersession

This round re-audits the same CivicCast SHA because the coder changed only the external PR description. Per the requested same-path procedure, this version of the canonical file supersedes its earlier Git versions at audit-control commits `bd8b05e7b0c70b0555a767901d73085a395359b7` and `16af38efefcf6d76f554b61706e5a84f74ab23f1`. Those commits remain preserved in history.

The round-5 request described CC-WS1-005 as the only open finding. Before this verdict, authoritative audit-control `main` had advanced to `16af38e`, which corrected the prior hash-filter premise and recorded CC-WS1-007. This audit independently reproduced that correction instead of relying on either the request or the intervening record.

## Scope and result

The requested PR-body SHA reconciliation passes:

- live PR head is exact `fafdee3bed292618cbccd6291efd3605081fcd0f`;
- base is `program/native-windows`, never `main`;
- the audit history now identifies `fafdee3b` as current;
- the evidence paragraph binds all four blobs to `fafdee3b`;
- the observer paragraph names the live `fafdee3b` pin; and
- read-only/no-release-authority language remains explicit.

The implementation, tests, committed blob identities, live checker, and observer authority boundary remain operationally sound. The verdict cannot become `AUDIT_PASS`, however, because the source evidence comment and refreshed PR history still assert the independently falsified claim that plain named-file `git hash-object` does not apply clean filters and therefore required `--path`.

## Severity rollup

- Blocker: 0
- Critical: 0
- Major: 1
- Minor: 0
- Nit: 0

## Finding

### CC-WS1-007 - Major - Evidence prose and repair history state false Git hash semantics

- **Dimension:** correctness / documentation / evidence integrity
- **Surface:** `.agent-runs/native-windows/ws1-release-truth/evidence/run_evidence.py:72-74`; live PR #287 audit-history paragraph
- **Evidence:** the source comment says plain `git hash-object <file>` does not filter and that CC-WS1-004 proved it. The PR history likewise says `hash-object needs --path for filter-aware blobs`. In a fresh disposable repository using Git 2.54.0.windows.1, `core.autocrlf=true`, `* text=auto`, and an asserted two-line CRLF working file, the index blob, plain named-file hash, `--path sample.txt sample.txt`, and `--path=sample.txt sample.txt` all returned `fbbee861521bd5355538b096fa3998541cd33909`. Only `--no-filters sample.txt` returned the raw-CRLF hash `17f2fc0a7500e6b218190262d5a329086ba965ff`.
- **Why it matters:** the explicit `--path` implementation is safe, but its claimed necessity and repair causality are false. That premise generated a false-positive canonical finding and an unnecessary source correction. Leaving the false statement in executable evidence code and the public audit history would preserve a known evidence-integrity defect at the gate intended to prevent proof drift.
- **Acceptance criterion:** at a new source SHA, correct the runner comment and PR audit history to state that plain named-file mode and explicit same-path `--path` mode are both filter-aware here, while `--no-filters` is raw. Commit a durable CRLF negative control that requires `plain == explicit-path == committed/index blob` and `--no-filters != committed/index blob`, then regenerate the evidence log last. Do not describe the safe explicit-path form as required to fix plain named-file filtering.

## Closed finding

- **CC-WS1-005:** closed. Every current-head, evidence-SHA, observer-pin, target-branch, and authority statement requested in round 4 is reconciled in the live PR body.

## Proofs re-run

- Live PR binding assertions -> head exact, base exact, open state, current `fafdee3b`, round-4 history present, evidence SHA exact, observer pin exact, and authority boundary explicit.
- `git diff --check 44463a197f326b7e607c31c56f28047ff1b00fb0..HEAD` -> exit 0 with no diagnostics.
- `python -m pytest tests/policy/test_release_truth.py -q` -> `22 passed in 3.54s`, exit 0.
- `uv run ruff check scripts/policy/check_release_truth.py tests/policy/test_release_truth.py .agent-runs/native-windows/ws1-release-truth/evidence/run_evidence.py` -> all checks passed, exit 0.
- Four evidence blob checks -> every log line equals both `git rev-parse HEAD:<path>` and `git hash-object --path <path> <path>`: checker `9c9cbb7ed334e394ff676eb833058c98e01923b8`, manifest `5ffae14585c7b067620acad0ada6db5b5063e744`, tests `92e95121949f6a6a3814ac8a6d1f5a6f2f716edc`, README `dbd62da5180732df7e1f9ece56fc6b6964296f2e`.
- Independent CRLF filter falsification -> `index == plain named-file == both explicit-path forms`; `--no-filters` differs, with exact hashes recorded above.
- Authenticated checker against the audited checkout README -> `release-truth: PASS`, exit 0.
- Authenticated checker against exact live CivicCast `main` README at `61e0449ecd5565889a711da5c6f9e1e1eb51af58` -> exactly `DRIFT: README.md never mentions the current release v1.0.0-rc14`, exit 1.
- Observer workflow at pre-verdict audit-control head `16af38efefcf6d76f554b61706e5a84f74ab23f1` -> exact `fafdee3b` pin, `contents: read`, `issues: write`, and no CivicCast release authority.
- Audit-control issue #3 -> remains open with the one known rc14/rc15 honest-red drift and coder annotation. The last authenticated hosted path remains run 29542897314; no duplicate hosted run was dispatched because this round did not change the checker or manifest.

## Audit-lite dimensions

- **Correctness and security:** checker behavior and read-only observer authority pass; the hash-command implementation is safe, but its explanatory contract is false.
- **UX:** no product UI changed. The public PR audit history accurately names the current SHA but inaccurately explains the repair.
- **Documentation:** current SHA/pin reconciliation passes; hash-semantics documentation fails.
- **Tests:** 22 policy tests pass. The durable named-file versus `--no-filters` CRLF regression required by CC-WS1-007 is absent.
- **Runtime:** authenticated developer-machine checks pass for the audited README and report exactly the known one-drift state for live `main`.
- **Escalation:** no full audit-team escalation is warranted; the remaining defect is bounded to evidence semantics and its public history.

## Confidence and boundaries

- **Class 1:** exact fresh detached SHA, full source/evidence state, live PR metadata/body, current governance history, observer pin, and authority trace.
- **Class 2:** 22 tests, three-file lint, diff hygiene, four blob bindings, and independent CRLF/filter falsification.
- **Class 3:** authenticated developer-machine live GitHub evaluation against audited and exact live-main README inputs.
- The current exact observer pin is verified statically; hosted authentication remains demonstrated by the prior run.
- The live-main rc14/rc15 mismatch remains honest-red external state, not a checker defect and not release authority.
- No service-session, hardware, clean-machine, or soak claim is part of this gate.
- No merge, tag, release, cutover, CivicCast branch edit, implementation edit, or PR-body edit was performed.

## Final rationale

`CHANGES_REQUIRED` is bound to `fafdee3bed292618cbccd6291efd3605081fcd0f`. Round 5 closes the PR SHA/pin reconciliation finding but independently confirms the currently authoritative CC-WS1-007 evidence-integrity finding. Because its acceptance criterion requires a source comment and committed regression change, closure will produce a new CivicCast SHA and require a new exact-SHA audit.

This verdict does not authorize merge or advancement. Scott Converse remains owner and tie-breaker. A future exact-SHA `AUDIT_PASS` can authorize merge only into `program/native-windows`, never CivicCast `main`, and never a tag or release.
