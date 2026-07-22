# Owner decision — WS5 supervisor contract approved as build basis (2026-07-21)

Decided by Scott Converse (owner). Banked 2026-07-20, recorded here after rc17
cleared the auditor thread (per the governance process: pre-ratification owner
state belongs in a `decisions/` record + auditor ratification before build).

## What is approved

1. **Contract.** `spec-supervisor.md` (D1–D9, ACs, halt triggers) is approved as
   the WS5 build basis.
2. **7-state machine (deviation from the spec's 5).** The spec enumerates
   `starting / ready / degraded / blocked_wsl_active / stopping`. The design adds
   two states, approved as designed:
   - `blocked_probe_unavailable` — required by the dual-runtime guard's decision
     table (guard D3: the probe-unavailable holding state is a mandated, tested
     condition the supervisor must be able to enter).
   - `maintenance` — the read-only health mode the installer and migration specs
     require the supervisor to run in while the maintenance interlock is held
     (children's read paths up, no workers, mutating endpoints refused).
   Rationale on record: both close cross-spec gaps, not scope creep.
3. **WS5-defined identity constants.** `SERVICE_NAME=CivicCastSupervisor`, the
   display name, and the event-log source (no spec named these).
4. **LocalSystem-for-beta.** No new decision; already accepted under ADR-0021.

## Acceptance-criteria change — "clean-machine proof" amended

Supersedes the charter/spec "two-pristine-machine proof":
- ~90% of the clean-machine proof runs in a Windows Sandbox / clean VM, and that
  is where the build starts. (Native has no WSL layer, so the nested-virtualization
  failure that broke the rc17 clean-VM run does not apply here.)
- Reboot-survival / session-0 supervisor proof needs a persistent clean VM (Sandbox
  is ephemeral) — reuse the proven session-0 recipe from program item 5.
- One soak test on a true separate bare-metal machine — deferred to a final beta
  candidate (8h, possibly 24h). The old "two pristine machines" requirement is retired.

## Gate status

Owner half of the WS5 gate is IN. The auditor half — Codex ratifies the 7-state
deviation and the contract as the build basis — is requested as the first auditor
task now that rc17 has cleared. Build begins on ratification.

**Scope note (the standing correction):** WS5 is one of TWO shipping products, not a
cutover. Per `2026-07-17-parallel-ship.md`, merging `program/native-windows` does not
retire the WSL line; WS5's docs describe one of two coexisting products.
