# Program status ledger â€” native-Windows migration

Live state of the charter Â§8 sequence. This file records outcomes with evidence
pointers; it never grants gates (verdicts in `verdicts/` do that) and never
edits the charter. Update on every gate event. Maintained by the coder;
challenged by the auditor; owned by Scott.

**Last updated:** 2026-07-17 (coder, overnight autonomous session â€” owner
directive: "work as autonomously as possible toward shipping the final beta
native Windows version"; owner-gated actions remain owner-gated).

| # | Item | State | Evidence |
|---|------|-------|----------|
| 0 | Bootstrap Phase 0 | **DONE** | AUDIT_PASS at `44463a19` (verdicts/bootstrap-phase-0/); PR #283 held open (main off-limits) |
| 1 | WS1 release-truth | **DONE** | AUDIT_PASS at `da2b273d` after 10 rounds; PR #287 merged to program/native-windows; observer live in this repo |
| 2 | WS2 Postgres restore drill | **IN AUDIT LOOP** | Round 1: CHANGES_REQUIRED at `3dda5fa7` (1 Critical + 2 Major, all functional). CI-executed at that head (run 29559102912 green; round-0 run 29557846920 failed and caught a real fixture bug). Fix round in progress: symmetric table compare, sequence state, constraints/indexes/grants, pg_dumpall globals artifact, run_full_drill end-to-end CI proof, dump-quiescence guard |
| 3 | Superseding ADR + claims-evidence rule | **SPECS AUDITOR-PASSED; ws3 implementation unblocked** | 13-round design review closed ALL SDR findings (reviews/2026-07-17-specs-design-review.md); ADR-0021 auditor half PASS at civiccast 3ca1d44d - OWNER MERGE PENDING; claims spec v13 is the implementation contract; trust-root pin + authority-record-v1 await owner acceptance |
| 4 | Dual-runtime exclusion guard | **SPEC AUDITOR-REVIEWED (v3, SDR-001 closed)** | Implementation next after ws3; WSL-side ExecCondition patch is owner-routed and a co-install prerequisite |
| 5 | Native supervisor + media-worker decision gate | **GATE HALF: PASSED** â€” supervisor build remains | Decision gate PASSED 2026-07-17: unmodified engine.py/worker.py natively â€” 4 hot swaps clean TS, 0.00s hard-kill reap + relaunch, 3 concurrent channels (civiccast `.agent-runs/native-windows/spike-decision-gate/`, commit `3b4b2451`). **Python media worker RETAINED; Rust gstreamer-rs port CLOSED** (recorded fallback). **Session-0 spike PASSED** 2026-07-17: engine as LocalSystem service, started 12s after boot, 6m14s / ~230MB broadcast before first interactive logon (Security 4624 primary source), 60 swaps, clean teardown (civiccast `.agent-runs/native-windows/spike-session0/`, commit `115531df`). Remaining Â§7 slice items: supervisor subsystem, packaging closure, two-pristine-machine proof |
| 6 | Hardware + live-peer + soak proofs | **BLOCKED ON 5 + LPM calendar** | Owner testing policy (decisions/2026-07-17-testing-policy.md): max 4h soaks; ALL real-world/hardware at LPM; this machine is software-only |
| 7 | LPM cutover + rollback rehearsal | **OWNER-GATED, not started** | Parallel-ship decision (decisions/2026-07-17-parallel-ship.md): WSL+native ship alongside, feature-diverge, sunset on evidence |

## Standing constraints in force

- Owner decisions stay owner's: merge to `main`, tags, releases, rc-line actions, LPM cutover, macOS start, NATS replacement.
- Severity calibration (decisions/2026-07-17-severity-calibration.md): only functional defects block audit rounds; prose nits batch as Minor.
- Fable capability window closes ~2026-07-19: coder model degrades to Opus 4.8. Mitigation: all remaining design/judgment work is being converted to execution specs + worker briefs NOW (civiccast `.agent-runs/native-windows/specs/` on program/native-windows); the auditor (Codex) is unaffected by the cliff and keeps full gate authority.

## All existential unknowns: CLOSED

GStreamer native closure (50/51 factories + caption round-trip) Â· PyGObject via
pip (`gstreamer-bundle==1.28.5`) Â· engine behavior native (switching, recovery,
multichannel) Â· session-0 pre-login service broadcast. What remains is
engineering and proof, not discovery.
