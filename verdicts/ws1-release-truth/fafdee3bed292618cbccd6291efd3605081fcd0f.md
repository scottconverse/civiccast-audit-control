# WS1 Release-Truth Round-4 Re-audit Verdict

- **Verdict:** `CHANGES_REQUIRED`
- **Slice:** `ws1-release-truth`
- **Requested gate:** Charter section 8 row 1, scoped to manifest, read-only checker, evidence, and external observer
- **CivicCast repository:** `https://github.com/scottconverse/civiccast`
- **Audited SHA:** `fafdee3bed292618cbccd6291efd3605081fcd0f`
- **Merge base:** `44463a197f326b7e607c31c56f28047ff1b00fb0`
- **Prior audited SHA:** `bcd43a8868c1ce71840dbcc4a42ac365dec6c092`
- **Branch / PR:** `claude/ws1-release-truth` / CivicCast PR #287, targeting `program/native-windows`
- **Codex thread ID:** `019f6cfb-27e1-7240-82ba-b81aaca03ca3`
- **Audit date:** 2026-07-16 (America/Denver)
- **Auditor environment:** Windows 11 10.0.26200 x64; PowerShell; Python 3.14.5; Git 2.54.0.windows.1; `uv` 0.11.15
- **Execution posture:** `sandbox=danger-full-access`; `approval_policy=never` (non-interactive)
- **Detached worktree:** `C:\Users\scott\Documents\Codex\2026-07-16\i-m-interested-in\work\civiccast-ws1-fafdee3b-audit`
- **Worktree state:** detached at the audited SHA and clean before and after verification
- **Activity relay:** <https://github.com/scottconverse/civiccast/pull/287#issuecomment-4997710872>
- **Audit continuity:** after the host reboot, two auditor conversations briefly overlapped. The first published this path at audit-control commit `bd8b05e7b0c70b0555a767901d73085a395359b7`; the resumed audit independently falsified its hash premise. This follow-on canonical correction preserves that Git history and records the reproduced behavior.

## Scope and correction to the prior verdicts

The audit covered the complete `bcd43a88..fafdee3b` change and reconciled it with the full WS1 slice, live PR #287, audit-control observer pin, and current release metadata.

The sole finding in the canonical `bcd43a88` verdict was a false positive, and the initial `fafdee3b` verdict repeated it. On Git 2.54.0.windows.1, `git hash-object <named-file>` already applies the clean filters associated with that file's path. `--path <p> <p>` is valid and produces the same filtered blob when the supplied attribute path and named file are identical. Raw working-byte behavior requires `--no-filters` or an unqualified stdin mode. The historical commits remain inspectable; this corrected file and `drift-catalog/2026-07-16-bcd43a88-hash-filter-false-positive.md` supersede their factual premise.

Claude updated the PR body after `bd8b05e7`; its current SHA and observer pin are now reconciled to `fafdee3b`. The body still repeats the false claim that `bcd43a88` needed `--path`, so that statement is included in the remaining finding below.

## Severity rollup

- Blocker: 0
- Critical: 0
- Major: 1
- Minor: 0
- Nit: 0

## Finding

### CC-WS1-007 - Major - Evidence prose and causality are false even though the command is safe

- **Surface:** `.agent-runs/native-windows/ws1-release-truth/evidence/run_evidence.py:72-74`; commit `fafdee3b` body; live PR #287 audit-history paragraph
- **Evidence:** the code and public handoff say plain `git hash-object <file>` does not filter and that the prior audit proved `--path` was required. A controlled CRLF working-tree reproduction instead produced the same filtered blob for the index, plain named-file mode, and both `--path` forms; only `--no-filters` differed.
- **Consequence:** the runner's output is correct, but the evidence mechanism documents a false distinction and false repair history. That premise caused an invalid canonical verdict, an unnecessary corrective commit, and an initial second verdict that repeated the error. It is therefore material to the program's evidence-integrity gate.
- **Acceptance criterion:** correct the runner comments, evidence prose, and PR history to state that named-file mode and the explicit same-path `--path` form are both filter-aware here. Add a reproducible CRLF negative control that requires `plain == explicit-path == committed/index blob` and `--no-filters != committed/index blob`; regenerate the evidence log last.

## Commands and proofs re-run

- `git rev-parse HEAD` -> exact audited SHA; `git status --short` -> empty before and after.
- `python -m pytest tests/policy/test_release_truth.py -q` -> `22 passed in 2.73s`, exit 0.
- `uv run ruff check scripts/policy/check_release_truth.py tests/policy/test_release_truth.py .agent-runs/native-windows/ws1-release-truth/evidence/run_evidence.py` -> all checks passed, exit 0.
- `git diff --check 44463a197f326b7e607c31c56f28047ff1b00fb0..HEAD` -> no diagnostics, exit 0.
- Every blob recorded in `build-verification.log` equals `git rev-parse HEAD:<path>`: checker `9c9cbb7...`, manifest `5ffae145...`, tests `92e95121...`, README `dbd62da5...`.
- Committed evidence log -> 22 tests, live checker exit 0, `Overall: PASS`.
- Independent authenticated developer-machine live check against the audited manifest and README -> `release-truth: PASS`, exit 0.
- Observer workflow at audit-control `eea633b9` -> exact `CHECKER_PIN=fafdee3b`; permissions remain only `contents: read` and `issues: write`. A new hosted run was not dispatched because this commit changes only evidence-runner code/prose, not the checker or manifest.
- Live PR body after the initial verdict -> current SHA/evidence/observer pin now name `fafdee3b`, but the historical explanation still says `--path` was required.

Controlled CRLF reproduction in a disposable repository with `.gitattributes` set to `* text=auto eol=lf` and actual CRLF working bytes:

```powershell
git -C $tmp rev-parse ':sample.txt'
git -C $tmp hash-object sample.txt
git -C $tmp hash-object --path sample.txt sample.txt
git -C $tmp hash-object --path=sample.txt sample.txt
git -C $tmp hash-object --no-filters sample.txt
```

Results: index, plain named-file, and both explicit-path forms all returned `fbbee861521bd5355538b096fa3998541cd33909`; `--no-filters` returned `17f2fc0a7500e6b218190262d5a329086ba965ff`.

## Confidence classes and boundaries

- **Class 1:** exact detached SHA, full corrective diff, final code/evidence, live PR metadata/body, observer pin, and authority surface.
- **Class 2:** 22 tests, lint, diff hygiene, blob comparisons, and the independent CRLF/filter negative control.
- **Class 3:** authenticated developer-machine live release check against the audited checkout.
- The last hosted observer run remains prior-pin evidence; an exact-`fafdee3b` hosted run was skipped because the observer-executed checker and manifest bytes are unchanged.
- No service-session, hardware, clean-machine, or soak claim is in this gate.
- No merge, tag, release, cutover, CivicCast branch edit, or implementation fix was performed.

## Final rationale

`CHANGES_REQUIRED` applies only to `fafdee3bed292618cbccd6291efd3605081fcd0f`. The implementation, tests, hashes, live checker, observer pin, and now-current PR provenance fields are operationally sound. Advancement remains withheld because the evidence runner and public history are factually wrong in precisely the area that generated the prior invalid verdict. Correcting that narrative and committing the durable CRLF control is the smallest closure.

Scott Converse remains the only merge and advancement authority.
