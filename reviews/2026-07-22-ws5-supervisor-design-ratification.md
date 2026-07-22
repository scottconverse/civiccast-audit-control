# WS5 supervisor design-basis ratification review

**Review date:** 2026-07-22

**Reviewer:** Codex, native-Windows program auditor

**CivicCast subject:** `d03f97daaecb229b0cffc1836bd731f91df61487` on `claude/ws5-supervisor`

**Review type:** pre-implementation design ratification; not a slice audit, not a verdict, and no gate advancement

**Execution posture:** `sandbox=danger-full-access`, `approval_policy=never`

**Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-ws5-design-d03f97da` (clean; ignored `.venv` created to rerun the pure-core tests)

## Executive conclusion

**The seven-state vocabulary is ratified as coherent and complete.**
`blocked_probe_unavailable` and `maintenance` are not scope creep. The former is
already a mandatory, tested output of the merged dual-runtime coordinator; the
latter is the only state that represents the installer/migration contract's
distinct read-only health posture. No eighth lifecycle state is presently
required.

**The complete WS5 design is not ratified unchanged as the build basis.** Two
Critical lifecycle-contract defects and two Major mechanism-contract defects
remain. They are narrow enough for one design addendum, but building first would
move known freeze, exclusion, control-pipe, and shutdown defects into product
code. WS5 implementation is therefore **NOT UNBLOCKED against `d03f97da`**.

This is a design hold only. It is not `CHANGES_REQUIRED`, `BLOCKED`, or an
`AUDIT_PASS` slice verdict.

## Severity rollup

- Blocker: 0
- Critical: 2
- Major: 2
- Minor/Nit: 0 (wording-only items batched and non-holding)

## Ratified elements

### Seven-state vocabulary

The five supervisor states plus the two additions cover the distinct externally
observable service postures:

`starting / ready / degraded / blocked_wsl_active / blocked_probe_unavailable / maintenance / stopping`

- `blocked_probe_unavailable` closes the exact mismatch between supervisor D6
  and dual-runtime guard D3/AC9. Unknown A2 state must remain non-authorizing;
  folding it into `degraded` would incorrectly permit workers, while folding it
  into `blocked_wsl_active` would falsely claim positive WSL activity.
- `maintenance` closes the exact installer-D3/migration-D1 gap. It is neither a
  generic degradation nor a WSL refusal: database, broker, and control-plane
  read health must be available while application writers and workers remain
  disabled.

Evidence: [design state set and precedence](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/design.md#L7-L13), [guard D3/AC9](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L22-L35), [installer maintenance obligation](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L36-L65), and [migration freeze obligation](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-migration-contract.md#L10-L26).

### WS5-defined identities

`SERVICE_NAME=CivicCastSupervisor`, display name `CivicCast Native Supervisor`,
Event Log source `CivicCastSupervisor`, singleton
`Global\CivicCastSupervisorSingleton`, and control pipe
`\\.\pipe\civiccast-supervisor` are internally consistent, product-specific,
stable enough for installer/repair/uninstall contracts, and do not collide with
the separately named WSL product or runtime-owner mutex. They are ratified as
the WS5 identities.

### Egress work-directory acceptance criterion

The owner decision requiring `CIVICCAST_EGRESS_WORK_DIR` to resolve inside the
ACL-protected `ProgramData\CivicCast\data` tree is sound and binding. The WS5
implementation/evidence must make it an explicit acceptance row: inspect the
actual control-plane/worker environment, resolve the final path, and include it
in the AC-S4 ACL evidence. A config default or source-string assertion alone is
not sufficient.

### Amended clean-environment approach

The owner amendment is acceptable for WS5 if evidence is assigned to exact
venues rather than summarized as “~90%”:

1. fresh Windows Sandbox/clean-VM runs cover the software lifecycle matrix;
2. a newly provisioned persistent clean VM covers cold reboot, session 0, and
   no-login recovery that Sandbox cannot preserve; and
3. the separate bare-metal soak remains a later final-beta activity, not a
   prerequisite to start or pass the WS5 software slice.

This supersedes the old two-pristine-environment wording for WS5. The older
2026-07-17 testing policy still caps this program's soaks at four hours, while
the 2026-07-21 amendment mentions an 8-hour or possible 24-hour bare-metal soak.
Those statements are coherent only if the later soak is outside the present
software-program gate. If it is to become a WS5 or native-program acceptance
criterion, Scott must explicitly supersede the four-hour cap first.

## Functional findings

### WS5-RAT-001 — maintenance is a state name without an enforceable cross-process mode contract (Critical)

The design says maintenance starts PostgreSQL, NATS, and the control plane in a
read-only posture and starts no workers. But D2 deliberately leaves worker
ownership inside the control-plane egress daemon, and the proposed package
layout defines no argv/environment/config contract by which the supervisor
puts that process into maintenance, no fail-closed behavior if the process does
not recognize the mode, and no readiness response that attests mutating routes
and background writers are disabled. The pure `workers_permitted()` predicate
cannot stop workers owned by another process.

**Consequence:** an upgrade/migration health start can bring up the normal app
lifespan, worker automation, or mutating endpoints while the maintenance marker
is held. That violates the enforced-freeze contract and can invalidate the
backup/schema recovery point.

Evidence: [design maintenance claim and package layout](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/design.md#L11-L56), [worker ownership remains in the control plane](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-supervisor.md#L12-L27), and [recon's existing app-lifespan ownership trace](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/recon/r1-seam.md#L74-L81).

Acceptance to close:

- define one versioned, fail-closed supervisor-to-control-plane maintenance
  input;
- enumerate every mutating HTTP/background-writer/worker start surface it
  disables while leaving the required read health available;
- make readiness report and verify the effective mode, refusing the upgrade
  health gate if the mode is absent, unknown, or ignored; and
- add expected-red tests showing a normal-mode or mode-ignoring control plane
  cannot satisfy maintenance readiness and cannot create writes/workers.

### WS5-RAT-002 — maintenance exit discards a concurrent guard block (Critical)

The pure state machine encodes `maintenance > blocked_*` by ignoring guard
events while maintenance holds. The coordinator's own ordered decision table
also returns on the held interlock before reaching A1/A2/A3, so a lower-priority
block may not be surfaced until a fresh post-release evaluation. On
`interlock_freed`, the state machine unconditionally moves to `starting`. The
executed transition sequence at the reviewed SHA was:

```text
ready -> interlock_held -> maintenance
maintenance -> guard_block_wsl -> maintenance
maintenance -> interlock_freed -> starting
```

The same transition loss occurs for `guard_block_probe` if supplied while the
interlock is held. More generally, the design does not require an atomic fresh
guard decision before leaving maintenance or enabling the control-plane
writer/worker posture. P4 proves only that an already-recorded blocked state
cannot jump directly to ready; it does not preserve or re-sample a condition
masked by maintenance.

**Consequence:** if WSL becomes active or A2 becomes unavailable during an
upgrade, releasing the interlock can enter startup without carrying the active
non-authorizing condition. That is a dual-transmission/fail-open risk.

Evidence: [transition implementation](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/civiccast/native/supervisor/states.py#L107-L155), [current precedence tests](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/tests/native/test_supervisor_states.py#L112-L144), and [coordinator interlock-first decision order](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/civiccast/native/runtime_guard.py#L127-L176).

Acceptance to close:

- make interlock release synchronously consume a current guard decision before
  any writer-capable child/mode starts;
- route a positive/unavailable result to the corresponding `blocked_*` state
  and route only `start`/`start_degraded` authorization to `starting`;
- retain or re-sample the lower-priority condition so event ordering cannot
  erase it; and
- add exhaustive sequence tests for both blocked outcomes arriving before,
  during, and at interlock release.

No eighth state is needed; this is an exit-transition/input-atomicity defect in
the otherwise correct seven-state vocabulary.

### WS5-RAT-003 — the named-pipe “read tier” cannot perform the documented request/reply exchange as stated (Major)

The D7 pipe is duplex JSON request/reply, but its SDDL description grants
Authenticated Users “read” access. An unprivileged status client must write a
`status` request before it can read the response. Microsoft documents that a
duplex client needs compatible write access, and also warns that generic write
includes `FILE_CREATE_PIPE_INSTANCE`; therefore the design must name the
minimal individual client rights instead of saying only “read tier” or granting
`FILE_GENERIC_WRITE`.

**Consequence:** a literal implementation fails AC-N1 because the non-admin
client cannot send `status`; an overbroad implementation restores the pipe-
instance/squatting permission the revised contract is meant to withhold.

Evidence: [supervisor D7](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-supervisor.md#L95-L117), [design pipe server](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/design.md#L41-L49), and [Microsoft Named Pipe Security and Access Rights](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights).

Acceptance to close:

- specify the exact low-level pipe access mask for Authenticated Users that
  permits request write + response read but not instance creation; and
- make AC-N1 a real-pipe test under a non-admin token, including a negative
  `CreateNamedPipe`/pre-creation attempt, not only a pure command-tier test.

### WS5-RAT-004 — the contract claims graceful worker drain through a path that does not exist (Major)

Supervisor D5 says graceful control-plane shutdown drains channels through the
existing daemon shutdown path. Recon traces that path and finds the FastAPI
lifespan stops only the automation polling thread; it does not enumerate active
channels or send `stop`/`drain`. The design correctly makes the Windows pipe
`stop` verb primary and defines its acknowledgement/replay behavior, but it
still does not name the drain-all caller that invokes that verb for every live
worker before the control plane exits.

**Consequence:** ordinary service stop/upgrade falls through to Job Object
termination for active workers. That is containment, not the promised graceful
shutdown, and the associated stop semantics can remain unreachable production
code.

Evidence: [contract D5 worker statement](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/specs/spec-supervisor.md#L70-L83), [recon shutdown trace](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/recon/r1-seam.md#L69-L81), and [design stop policy](https://github.com/scottconverse/civiccast/blob/d03f97daaecb229b0cffc1836bd731f91df61487/.agent-runs/native-windows/ws5-supervisor/design/design.md#L59-L69).

Acceptance to close:

- add an explicit app-lifespan/daemon shutdown owner that snapshots all active
  channels, sends terminal stop, waits on process exit as ground truth, and
  escalates only after the bounded deadline; and
- prove multi-channel service stop, lost stop acknowledgement, one hung worker,
  and supervisor death separately so graceful behavior is not inferred from
  Job Object cleanup.

## Proof executed

- Bound a separate detached worktree to
  `d03f97daaecb229b0cffc1836bd731f91df61487`; it remained clean apart from an
  ignored `.venv` created by `uv`.
- Read the requested design, all four recon reports, supervisor contract,
  directly implicated guard/installer/migration specs, owner decisions,
  testing policy, charter/gate/protocol, and the committed pure-core source and
  tests.
- Re-ran the four pure supervisor suites:
  `59 passed in 0.89s`.
- Executed the maintenance/guard ordering counterexample shown in
  WS5-RAT-002.
- `git diff --check d03f97da^ d03f97da` passed.
- Reconciled the pipe concern against Microsoft's primary Win32 documentation.

## Evidence boundary and source-control actions

Confidence is exact-SHA static design review plus execution of the committed
pure core and one adversarial state sequence. No Windows service, real named
pipe, Job Object, child process, session-0, clean-environment, installer,
hardware, live-peer, or soak proof exists or is claimed by this review.

No CivicCast source or coder-branch file was modified. This review is the only
audit-control artifact added. Scott remains owner and tie-breaker; this record
grants no merge, tag, release, cutover, or slice-gate authority.
