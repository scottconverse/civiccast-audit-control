# rc17 Beta-Blocker Salvage Round-6 Audit Verdict

- **Verdict:** `AUDIT_PASS`
- **Slice:** `rc17-beta-blockers`
- **Requested gate:** exact-SHA closure review for round-5 finding CC-RC17-006
- **CivicCast repository:** `https://github.com/scottconverse/civiccast`
- **Audited SHA:** `d1f5d9c0c2bef750b1e749aee40a5cdf7eed2b69`
- **Audited parent:** `d0ab883835be53b83df0025ad76e89485d849d9c` (round-5 `CHANGES_REQUIRED`)
- **Current `main` merge base:** `338fb332c1010f3229e59d6acae2c79c9043a7c2`
- **Branch / PR:** `fix/rc17-beta-blockers` / CivicCast PR #299, ready for review, targeting `main`
- **Codex thread ID:** `019f7e3d-242d-7561-b07e-c23fc9f87059`
- **Audit date:** 2026-07-20 (America/Denver)
- **Auditor environment:** DESKTOP-VBMA6O5; Microsoft Windows 11 Pro 10.0.26200 x64; Windows PowerShell 5.1; Python 3.12.10; uv-managed Python 3.13.14; pytest 9.0.3; Git 2.54.0.windows.1; uv 0.11.23; ruff 0.15.12
- **Execution posture:** `sandbox=danger-full-access`; `approval_policy=never`
- **Detached worktree:** `C:\Users\Scott\Desktop\CODE\_audit-worktrees\civiccast-rc17-beta-blockers-d1f5d9c0-r6`
- **Worktree state:** exact, detached, and clean at binding and verdict writing
- **Activity relay:** <https://github.com/scottconverse/civiccast/pull/299#issuecomment-5019697425>
- **Owner scope:** release-prep exact-SHA continuity; this source pass does not itself authorize or prove the tag/public artifacts

## Executive result

The single-file change closes CC-RC17-006. The release-posture guard now groups Markdown source into logical blocks separated by blank lines and list items, strips blockquote prefixes, and matches forbidden prior-release directives against normalized block text. Ordinary line wrapping no longer bypasses the control, while paragraph and list-item boundaries do not fuse unrelated fragments.

The exact round-5 auditor falsifications now return four violations across four test surfaces, including an independently supplied wrapped blockquote. The exact round-4 parent still returns exactly 11 violations across the six originally cited files, and the corrected head returns zero. The focused file returns 5 passed; the broader documentation/policy blast radius returns 329 passed with one documented environment skip. No product, runtime, or public-document content changed in this round.

No finding remains in round-6 scope. `AUDIT_PASS` binds to this exact source SHA and preserves the owner-recorded partial proof boundary: it is not evidence for a tagged/public artifact or the later exact-download clean-host lifecycle walkthrough.

## Severity rollup

- Blocker: 0
- Critical: 0
- Major: 0
- Minor: 0
- Nit: 0

## Findings

No new finding. CC-RC17-006 is closed.

## Closure evidence

- `_blocks()` at `tests/policy/test_release_posture_consistency.py:99` ends blocks at blank lines and starts a new block for each list item, matching the established candidate-boundary model.
- `_clean()` removes one or more blockquote prefixes and surrounding whitespace; `_block_text()` joins cleaned source lines with single spaces.
- `evaluate_release_posture_consistency()` now searches the normalized logical block and retains a source location, using the containing block's first line for a wrapped phrase.
- `test_guard_goes_red_on_reflow_wrapped_directives` covers `is the / current`, `Use only / tag`, and blockquoted `is / unpublished` splits.
- `test_guard_does_not_join_across_blank_lines_or_list_items` protects the false-positive boundary.
- Auditor rerun of the round-5 temporary fixtures returned `WRAPPED_VIOLATIONS=4` for README, FAQ, SUPPORT, and an additional blockquoted ARCHITECTURE variant. The same run returned `SEPARATED_VIOLATIONS=0` for paragraph, list-item, and blockquoted paragraph separation.
- Direct evaluation against detached round-4 worktree `a574645a` returned exactly the expected 11 violations. Direct head evaluation returned zero.

## Scope and proofs executed

- Live PR inspection bound PR #299 to exact head `d1f5d9c0c2bef750b1e749aee40a5cdf7eed2b69`, base `main`, ready-for-review state, and mergeable status.
- The complete one-commit delta changes only `tests/policy/test_release_posture_consistency.py`: 103 insertions and eight deletions. No documentation or product/runtime file changed.
- `uv run pytest -q tests/policy/test_release_posture_consistency.py`: 5 passed in 1.01 seconds.
- `uv run pytest -q tests/policy tests/docs tests/test_audit_protocol_docs.py`: 329 passed, one documented environment skip (`systemd-analyze` is available on the Linux cleanroom runner).
- `uv run python scripts/policy/check_release_identity.py`: PASS for `v1.0.0-rc17`.
- `uv run python scripts/policy/check_release_candidate_boundary.py`: PASS.
- `uv run python scripts/render_user_manual.py --out-dir docs --check-current`: PASS.
- Ruff check and format check on the changed file: PASS. `git diff --check d0ab8838..HEAD`: PASS. Final detached worktree: clean.
- `gh pr checks 299` at verdict time: all 17 exact-head jobs passed, including the 15m04s Docker cleanroom and 22m43s Python 3.12 unit job.

## Five-dimension result

- **Correctness and security:** test-only delta; block normalization fixes the proved bypass without changing product/security behavior.
- **UX:** no UI changed; the already-accepted rc17 public directions remain unchanged and consistent.
- **Documentation:** no doc changed; current public surfaces remain green under the strengthened guard.
- **Tests:** exact historical failures, wrapped directives, legal historical language, and non-fusing boundaries are all exercised and pass at head.
- **Runtime:** no product runtime path changed. Exact-head hosted CI is reported above; source/CI evidence is not final-public-artifact proof.

## Confidence and boundaries

- **Class 1:** exact detached checkout, complete one-file diff/final-state review, PR identity, and live CI inspection.
- **Class 2:** focused and broader tests, exact-round-4 evaluation, adversarial wrapped/separated fixtures, release identity/boundary checks, manual render, ruff/format, and diff hygiene.
- **Not established by this round:** built/tagged rc17 artifact identity, hashes/signatures, live release URLs/assets, cache-busted visitor audit, or exact-public-installer clean-host lifecycle proof.
- No CivicCast source edit, coder-branch edit, merge, tag, release, artifact publication, clean-machine installation, LPM action, or public-release change was performed by the auditor.

## Final rationale

The matcher now enforces the intended semantic boundary rather than a physical-line accident. It catches the exact historical defect and the round-5 reflow counterexamples, respects paragraph/list separation, leaves legal historical language untouched, and passes the full relevant blast radius. No functional finding remains.

`AUDIT_PASS` binds only to CivicCast SHA `d1f5d9c0c2bef750b1e749aee40a5cdf7eed2b69`. It closes CC-RC17-006 and restores exact-SHA audit continuity through the rc17 release-prep head. It does not validate or authorize a tag, public release, public artifact, or LPM handoff.
