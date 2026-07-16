# CivicCast Native-Windows Audit Protocol

## 1. Purpose and authority

This repository is the authoritative governance record for the native-Windows program. `CHARTER.md` defines scope and gates; `AUDIT_GATE.md` defines advancement; this document defines the coder-to-auditor handoff and canonical verdict record.

Roles:

- Claude is the coder.
- Codex is the auditor and verdict author.
- Scott Converse is owner and tie-breaker, and the only authority for CivicCast
  `main`, tags, releases, and cutover decisions. This protocol records his
  standing authorization for exact-SHA `AUDIT_PASS` slice merges into
  `program/native-windows` only.

Neither agent certifies its own work. CivicCast repo instructions may summarize these rules but cannot redefine them.

## 2. Trust model

The MCP channel and the shared GitHub PAT do not authenticate Claude versus Codex. This repo is a **process boundary, not a credential boundary**. A request can ask for an audit; it cannot manufacture authority. A verdict is credible only when its exact source SHA, Codex thread ID, execution posture, evidence, and rerunnable results agree.

History is inspectable but not protected from every actor on this machine. Unexpected edits to this repo are themselves governance drift and must be recorded in `drift-catalog/` and resolved by Scott.

### Native-Windows integration and release boundary

Effective with the Bootstrap Phase 0 pass at CivicCast SHA
`44463a197f326b7e607c31c56f28047ff1b00fb0`:

- CivicCast `main` is off-limits to native-Windows program work. The WSL rc line
  and LPM beta remain owned by Scott and the rc-line coder.
- Native-Windows work integrates only into `program/native-windows`, initialized
  from the audited SHA above. Slice PRs target and merge into that branch only
  after an exact-SHA `AUDIT_PASS`. Native-Windows work creates no tags.
- CivicCast PR #283 stays open and held as the eventual `main`-landing vehicle;
  only a future explicit instruction from Scott can authorize that landing.
- Audit-control `main` remains the live canonical verdict ledger.
- The WS1 release-truth checker is a read-only external observer of CivicCast
  releases. It may open drift-alarm issues, but it has no authority over the WSL
  release line. In particular, rc15 ships only on Scott's and the rc-line coder's
  authority.

## 3. Audit request contract

The coder request must contain:

- `slice`
- source repository and branch
- `base_sha` and exact `head_sha`
- requested charter gate
- enumerated claims and acceptance criteria
- repo-relative evidence paths
- exact commands already run and their results
- reproducible negative controls and expected-red conditions
- known gaps
- requested `sandbox` and `approval_policy`
- requested worktree posture
- PR or issue URL used as the public activity relay

Ambient settings are not evidence. The auditor records the actual per-call sandbox and approval policy in the verdict. If the requested posture cannot execute a required proof, the auditor changes it only within owner-authorized scope and records the change, or returns `BLOCKED`.

## 4. Checkout and scope binding

Before reading conclusions or executing tests, the auditor:

1. Resolves the live PR/repository identity and confirms `head_sha` still names the requested commit.
2. Creates a separate detached worktree at that SHA.
3. Records repository URL, full SHA, merge base, branch/PR metadata, and `git status --short`.
4. Stops if the supplied SHA differs from the live request or if the worktree cannot be isolated.

The coder's working tree is never the audit environment. Worktree proof must not be reported as clean-machine proof.

## 5. Evidence execution

The auditor performs all applicable checks:

- Read the complete scoped diff and the final cross-file state.
- Trace changed contracts through callers, tests, docs, and runtime entry points.
- Re-run every named negative control and material positive proof that is safe and available.
- Verify that tests can fail for the condition they claim to detect.
- Validate evidence schemas, source SHAs, commands, tool versions, exit codes, timestamps, hashes, and provenance.
- Reconcile README, changelog, PR body, release state, and public surfaces when they are in scope.
- State skipped or unavailable checks and the exact confidence boundary they create.

The coder's passing output is an input to the audit, not the auditor's result.

### Audit activity visibility

The MCP auditor runs in a separate headless conversation and cannot write directly
into Scott's already-open Codex Desktop task. Silence in that task must therefore
never be used as an audit status signal. The auditor emits explicit milestones to
the request's public PR or issue, and a Codex Desktop thread heartbeat relays those
events into Scott's task.

Required milestones:

1. Post `AUDIT_STARTED` immediately after binding the detached worktree. Include
   slice, full source SHA, Codex thread ID, actual sandbox/approval posture, and
   the first proof phase.
2. Update one activity comment at every phase transition. Use the phases
   `CHECKOUT_BOUND`, `STATIC_REVIEW`, `FALSIFICATIONS`, `RUNTIME_PROOF`, and
   `VERDICT_WRITING` as applicable. State the last completed check and the next
   check.
3. Before any command expected to run longer than approximately two minutes,
   announce the command and expected duration. Update the activity comment
   immediately after the command returns with its observed outcome.
4. Post `AUDIT_FINISHED` with the exact verdict, source SHA, canonical verdict URL,
   and audit-control commit.

Milestones are operational telemetry, not verdicts and not authority. Keep one
updatable activity comment per audit thread to avoid PR noise. Between actions,
60 seconds is a best-effort visibility target, not a conformance bound; the model
cannot emit while a single command or tool call is still running. A heartbeat must
not speculate about results. Under the security-embargo lane, publish only the
phase and `OWNER_CONTACT_REQUIRED`; never expose finding details.

If the PR/issue relay is unavailable, rate-limited, or rejects an update, the
auditor records `relay unavailable from phase <phase>` in the canonical verdict
and continues the audit. Relay failure does not abort, block, downgrade, or
invalidate the audit or verdict. When practical after recovery, update the same
activity comment with a concise summary of the missed phase transitions.

## 6. Confidence classes

Every runtime claim is labeled at the strongest layer actually executed:

1. Static/source inspection
2. Unit or integration test in the detached worktree
3. Developer-machine runtime
4. Session-0 Windows Service
5. Physical hardware or live-peer execution
6. Independently pristine Windows machine
7. Duration/soak proof

Higher classes do not arise by inference. In particular, VM reset, container isolation, and clean Git state do not establish class 6.

## 7. Verdict record

Canonical verdict path:

`verdicts/<slice>/<source-sha>.md`

Each verdict records:

- verdict token: `AUDIT_PASS`, `CHANGES_REQUIRED`, or `BLOCKED`
- CivicCast repository URL and full audited SHA
- merge base and PR number, when applicable
- slice and charter gate
- Codex thread ID
- audit date and auditor execution environment
- actual sandbox and approval policy
- detached-worktree path and dirty state
- claims checked
- commands and negative controls re-run, with results
- findings by severity and file/line
- confidence classes achieved
- known gaps and skipped checks
- final rationale

The verdict is committed in this repository. Its audit-control commit SHA is reported to the coder and owner.

Verdicts never carry across source SHAs. Rebase, amendment, evidence-pointer commit, or any other source change requires a new audit of the new SHA.

## 8. CivicCast-side pointer contract

The canonical verdict lives here. A CivicCast-side pointer is only a cache/reference and cannot be the source of pass authority.

To avoid the self-reference loop created by committing a `verdict-<HEAD>.txt` after auditing that HEAD, the completion gate must not treat a newly committed pointer as proof for its own new SHA. The supported designs are:

1. The hook resolves `verdicts/<slice>/<current-head>.md` from an audit-control checkout or GitHub and validates the canonical record; or
2. A local, uncommitted pointer cache names that canonical verdict and is independently validated by the hook.

If a local pointer cache is used, it contains exact structured fields, not only a status prefix:

```text
verdict=AUDIT_PASS
source_sha=<40-hex CivicCast SHA>
thread_id=<Codex thread ID>
verdict_repo=https://github.com/scottconverse/civiccast-audit-control
verdict_path=verdicts/<slice>/<source-sha>.md
audit_control_commit=<40-hex governance commit SHA>
```

The hook must compare the current CivicCast HEAD, slice, thread ID, canonical verdict content, and audit-control commit. A string beginning with `AUDIT_PASS` is not sufficient. Network or audit-control lookup failure fails closed for a program-slice completion.

Any CivicCast task-selection convention must use fields actually delivered by the installed Claude Code `TaskCompleted` event. As of the founding protocol, the documented event provides `task_id`, `task_subject`, optional `task_description`, `teammate_name`, and `team_name`; it does not provide arbitrary task metadata. Tests use a captured or documentation-exact event payload.

## 9. Security-embargo lane

Do not put exploitable details, credentials, private host data, or an unpatched vulnerability recipe in this public repository or a public PR.

When such a finding appears:

1. Stop public detail publication.
2. Notify Scott through the approved private channel with the exact SHA and a minimal description.
3. Store detailed evidence only in the private location Scott designates.
4. Do not commit a public finding or verdict while the embargo is active.
5. After remediation and owner approval, audit the new SHA. Publish the normal verdict and only the historical disclosure detail Scott authorizes.

Security embargo does not mean pass, and a fixed descendant SHA never inherits the embargoed SHA's verdict.

## 10. Findings and disagreement

Findings identify severity, exact file/line or runtime surface, evidence, consequence, and the smallest acceptance criterion that closes them. The auditor does not implement fixes in the audited branch.

If coder and auditor disagree, both positions and evidence are preserved and Scott decides. A tie-break changes neither history nor the exact-SHA binding; it records an owner decision.

## 11. Drift catalog

The drift catalog tracks mismatches among CivicCast pointers, canonical verdicts, PR status, release state, and these governance files. A catalog entry names the observed state, expected state, detection command, owner, disposition, and closing evidence. Empty means no drift has been recorded, not that a checker has proved no drift exists.

## 12. Bootstrap genesis

The initial audit-control commit and CivicCast PR #283 are Bootstrap Phase 0. Because the MCP loop and canonical gate do not exist until those changes land, the genesis uses manual cross-review:

- Codex seeds this repo and reviews PR #283.
- Claude reviews the seed commit.
- Scott approves both halves.
- Nothing merges into CivicCast and no program workstream advances until Scott approves.

Bootstrap Phase 0 subsequently received `AUDIT_PASS` at CivicCast SHA
`44463a197f326b7e607c31c56f28047ff1b00fb0`. Scott's current integration
instruction is recorded under **Native-Windows integration and release
boundary** above: new slices advance through `program/native-windows`; PR #283
remains held and the pass does not authorize a CivicCast `main` merge.
