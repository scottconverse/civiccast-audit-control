# CivicCast Native-Windows Program Charter

**Date:** 2026-07-16
**Status:** Proposed - consolidates the seven-round Claude/Codex review of 2026-07-16 into one work order.
**Scope:** CivicCast primary; CivicSuite carries a short companion punch list (section 9).
**Decision being chartered:** Retire the WSL2 deployment architecture on Windows in favor of a native Windows runtime, via a superseding ADR and an incremental, proof-gated migration. Keep the WSL line operational until the native path reaches equivalent operational proof.

---

## 1. The decision and why it is now justified

ADR-0003 (2026-05-08) rejected native Windows deployment (Option B) because it "requires a permanently forked code path" whose "benefit is marginal given WSL2's ubiquity," and costed WSL2 as "a single-file download with one prompted reboot." Both premises are now falsified by evidence:

1. **The WSL cost model inverted.** Thirteen release candidates, a per-user WSL keepalive host (`civiccast/apps/installer/src-tauri/src/main.rs` - `wsl.exe ... sleep infinity` + `systemctl restart` recovery), and the rc13 withdrawal ("a genuine clean-Windows test later exposed bootstrap failures that the earlier contaminated lab run did not catch") are the measured price of the "marginal benefit" trade.
2. **The required media-pipeline closure was not Linux-bound.** All 51 factories classified as required for CivicCast's current and planned playout graphs exist in the official GStreamer 1.28.5 MSVC Windows runtime, including the previously omitted `souphttpsrc`. The code-reachable inventory is broader and is intentionally classified: five profile-selected encoders are conditional (`x265enc`, `nvh264enc`, and `nvh265enc` were present; Linux-only `vah264enc` and `vah265enc` were absent and require a Windows guard/remap), while 11 hardware decoders are rank-demoted opportunistically and their absence is tolerated by design (5 present, 6 absent on the developer box). The caption-SEI embed leg (`cccombiner`/`h264ccinserter`) ran natively end-to-end with an ffmpeg-subcc decode-back returning the exact injected text. The Rust `gst-plugin-closedcaption` crate the Linux runtime compiles from source ships in-box on Windows. Classified manifests and executable proof live in CivicCast PR #283 at `.agent-runs/native-windows/spike-gstreamer-bundle/evidence/` at `e45b92d9b68c13d5ab5cd8a08caeeb9ce0e15301`; the earlier installer report is `C:\Users\scott\Desktop\CODE\civiccast-windows-native-gstreamer-verification.md`.
3. **Native Windows is required by the roadmap, not just preferred.** DeckLink is PCIe; WSL2 has no PCIe passthrough, so the readiness ledger's SDI/headend rungs are unreachable from the current architecture. The official Windows runtime additionally exposes hardware paths WSL never can (`d3d12h264dec`, `mfh264enc`, Media Foundation).

ADR-0003's own footer requires supersession by a new ADR. That ADR (WS-2 below) must record both falsified premises **and** honestly account the costs Option B was rejected for - Windows service management, path handling, a separate installer surface, an expanded CI matrix - as now-accepted costs, plus the session-0 obligations in section 5.

**Honesty boundary:** the architectural *direction* is evidence-backed. The *implementation* proofs - session-0 hardware access, Postgres restore, dual-runtime exclusion, clean-machine packaging, long-duration Windows playout - are open and enumerated in section 7. Nothing in this charter claims them in advance.

---

## 2. Target architecture

```text
Tauri/React operator console  (restartable at will; never load-bearing for playout)
        |  localhost API
Python/FastAPI control plane  (scheduling, domain logic, DB - unchanged)
        |  versioned narrow command/event protocol
Native media worker decision gate
        |-- first candidate: existing Python/PyGObject engine via gstreamer-bundle
        `-- conditional fallback: Rust/gstreamer-rs worker if the evidence requires it
Native media/data plane: GStreamer (official MSVC runtime, minimal closure),
FFmpeg, TSDuck, PostgreSQL, NATS - supervised by a session-0 Windows Service
```

Key choices, all settled in review:

- **Keep Python/FastAPI and TypeScript/React.** The languages were never the problem; no rewrite.
- **Make the Rust media-worker port a time-boxed decision gate, not a predetermined task.** The existing spec-driven Python engine is the first candidate because official `gstreamer-bundle==1.28.5` now provides native CPython/PyGObject and GStreamer wheels on Windows. The committed spike in CivicCast PR #283 at `.agent-runs/native-windows/spike-gstreamer-bundle/evidence/` proves native CPython 3.12 loading, 51/51 required factories, the classified conditional and absence-tolerant results above, a production-shaped caption pipeline to EOS, and verbatim caption decode-back. The gate remains open until the two Linux-only encoder mappings are guarded/remapped and source switching, recovery, multichannel operation, and session-0 execution are proved. Compare the Python and Rust candidates on those behaviors, packaging/license closure, memory/soak stability, service integration, maintainability, and implementation cost. Select Rust only if measured evidence shows the Python candidate cannot meet the production bar or Rust materially reduces lifecycle risk.
- **Do not preselect the worker IPC transport.** The command/event contract must be versioned, authenticated, authorized, and proven across service restart; named pipes versus loopback is a slice decision.
- **Session-0 Windows Service owns the media plane.** The current HKCU `Run` + per-user keeper means playout has no guaranteed operation independent of a user session. The service on macOS is launchd; on Linux, systemd - one supervisor contract, per-OS adapters.
- **Tauri is the console, not the keeper.** UI restart must not interrupt playout.
- **Keep NATS** unless measurements show operational cost; replacing it requires its own ADR (ADR-0001 chose it deliberately; CivicSuite ADR-0009 is the in-family precedent for a Postgres-queue answer if that day comes).
- **NDI/SDI egress relays stay BYO-ffmpeg** (`-f libndi_newtek`, `-f decklink` per `civiccast/egress/ndi_relay.py`, `automation.py`). NDI licensing is **dormant, not moot**: it activates if CivicCast ever bundles NDI components, ships a GStreamer runtime containing an NDI plugin, or internalizes the relay. Trademark/attribution obligations may apply even under BYO - check once, record in the ADR.

The broad `gstreamer-bundle` meta-package and its GPL tiers are spike inputs, not authorization to ship the full bundle. Production still requires the minimal, checksum-locked, license-reviewed plugin closure described in section 7.

---

## 3. Workstream 1 - Release truth (do first; cheap; both repos)

A recurring cause of factual errors in the review - in the repo docs and in both reviewers - was drift between README/docs and live release state (rc13 presented as current after withdrawal; CivicSuite README claiming signing 1.0.2 didn't have; 1.0.4 marked Latest with a placeholder body).

1. **A committed expected-releases manifest is the sole authored source** for release state: tag -> status (current/withdrawn/superseded) -> current-pointer -> asset names+hashes -> signing status. README release sections are **generated** from it, never hand-edited.
2. **Extend the existing live checker** - `scripts/policy/check_published_release_assets.py` already reads `gh release view` (draft state, tag, prerelease flag, assets). Add: WITHDRAWN/supersession scan of release title+body, and manifest-vs-live diffing. A deleted release is then an ordinary diff against the manifest, not a special case the checker can't inspect.
3. **Real triggers - in a separate `release-truth` audit workflow, not `release-artifacts.yml`.** The build workflow runs on `workflow_dispatch` and `v*` tag pushes only, so release-page edits currently fire nothing; but wiring `release` events into *that* workflow could rebuild or re-upload artifacts when someone edits or deletes a release. The audit workflow gets the `release` event trigger (`published`, `edited`, `deleted`) plus a daily cron backstop, runs the manifest-vs-live diff only, and holds read-only (`contents: read`) permissions.
4. Record in the manifest, once: whether CivicSuite 1.0.3 was the first signed release (open historical question; requires binary signature verification of the 1.0.3 MSI).

## 4. Workstream 2 - Claims-evidence rule + superseding ADR

Two mechanisms, deliberately separate:

- **The manifest (WS-1) governs release state.** It cannot police prose.
- **A capability-claims rule governs prose claims** ("implemented," "proven," "validated") in README/docs. The seed exists - `scripts/roadmap_status.py` - but it checks evidence *existence* only (`file_exists`, `migration_exists`): a test file's presence counts as proof. Extending it is substantial new verification logic, not schema work. Full-strength rule per claim:
  - code pointer **and** an executable test bound to the claim;
  - a **reproducible negative control** - a falsification bound to source commit, exact command, environment, and result; CI re-runs it where possible;
  - **commit-bound proof artifacts** with validated schema: source SHA, command, environment/tool versions, exit status, timestamps, hashes, provenance. An unvalidated proof file is `file_exists()` with extra steps. `docs/releases/evidence/` is the existing home/format; the project's Sigstore machinery can attest artifacts CI cannot re-run (hardware/manual evidence). Attestation proves provenance and integrity, not that a manual observation was truthful; raw observations and review remain required.

  Motivating example from this review: `pg_restore` had a "pointer" - it pointed at a docstring (`civiccast/dr/__init__.py:26`), and the DR verification doc's "Postgres backup/restore is implemented" was wrong on its restore half - an overclaim inside the honesty machinery itself.

- **The superseding ADR for ADR-0003** records: both falsified premises (section 1), the accepted Option-B costs, the session-0 requirement, the conditional media-worker decision gate (section 2), the dual-runtime exclusion design (section 6), the NDI licensing posture, the BYO-ffmpeg egress split, the NATS position, and the migration/cutover contract (section 7). ADR-0003's Status flips to Superseded with the one-line pointer, per its own footer.

## 5. Workstream 3 - Postgres restore drill (owed to the *current* product)

Finding: the executed 0.5.0 DR drill was **SQLite** (92 tables via Alembic against a fresh SQLite file). Postgres `pg_dump` is real and CI-exercised (Postgres 17 testcontainer, in-container dump, artifact exists + nonempty), but **no `pg_restore` path exists anywhere in the repo**, and `civiccast/dr/report.py` dispatches only the SQLite restore drill; `cli.py:2267` rejects non-SQLite drill URLs. Production runs Postgres, so the readiness ledger's DR row and the cold-standby claim are currently supported only at their database *shape*, not their database *engine*.

Sequencing (one implementation, two distinct proofs):

1. **Build the restore drill against the shipping WSL product first.** This retires the standing DR overclaim on its own timeline, independent of any migration decision.
2. **Re-execute the shared machinery against native Windows PostgreSQL** as a migration gate. "Shared machinery" = one shared restore contract with platform-specific execution adapters (`pg_restore` discovery, path translation, credentials, service accounts, DB ownership/roles all differ). Both executions pass independently; neither inherits the other's proof.

Verification depth: confirm source and restored databases share the same Alembic revision (precondition), then **introspect and compare** application-owned tables, sequences, constraints, extensions, roles/permissions, and data (proof). Table count is an observed metric, never a fixed assertion. The SQLite drill's snapshot-compare harness (`snapshot_tables()` re-run against the restored copy) is the pattern; Postgres adds object classes SQLite doesn't have - that delta is the real work. Restore drill closes the **database portion** of three claims (DR row, cold standby, cutover); media, config, certs/secrets, Windows path conversion, Windows service identity, ownership transfer, and rollback remain separately owed (section 7). (PostgreSQL roles/permissions and extensions belong to the restore proof above; the migration-side "identity" item is the Windows service account - a different thing.)

## 6. Workstream 4 - Dual-runtime exclusion

Hazard: during any period where both the WSL install and the native install exist on one machine, both could transmit. Design (host-owned, not database-owned - a database copied during migration is two databases afterward, each happy to bless its own runtime):

- **Authoritative selector:** `active_runtime = wsl | native`, machine-global, Windows-host-owned, enforced by the session-0 supervisor. Any DB marker is advisory display only.
- **Bidirectional refusal:** native refuses if the WSL service is running/healthy; WSL refuses if the native service owns the station.
- **Cutover:** disable and stop the WSL service, remove the HKCU `Run` keeper entry (from every profile that registered it - the resurrection risk is the installing user signing back in, or multiple registered profiles), set `active_runtime=native`, leave the distro intact for rollback.
- **Rollback:** stop native first, then re-enable WSL. The rollback contract must define what happens to data created *after* native activation: either a reverse migration path, or an explicit recovery-point/data-loss boundary stated to the operator at cutover time.

## 7. Workstream 5 - Native vertical slice, migration, and the proof ladder

**Vertical slice** (build order within the slice):

1. Session-0 native supervisor: install and **validate** the bundled runtimes (GStreamer minimal closure, FFmpeg, TSDuck) and **supervise** the executable services - PostgreSQL, NATS, the Python city services, and the selected media worker. (GStreamer is a library the worker loads, not a child process; its "supervision" is startup element validation, as the doctor lane does today.) The rc13 host's retry/backoff/logging concepts carry over; supervising native children is a new subsystem, not a port.
2. Complete the **time-boxed media-worker decision gate** from section 2. Through the native Python candidate, reproduce the caption pipeline and prove source switching, recovery, multichannel operation, and session-0 execution. Compare the named Python/Rust criteria and select the implementation from evidence. If the Python candidate passes the production criteria, retain it; if it fails or Rust materially reduces demonstrated lifecycle risk, implement the thin Rust `gstreamer-rs` worker while preserving the existing `ElementSpec` graph contract.
3. Play one scheduled asset continuously; kill each process and verify recovery; reboot and verify unattended resume (no user sign-in).
4. Exercise one BYO-ffmpeg NDI relay and one physical DeckLink path (plugin presence is proven; driver/device operation is not).
5. Three concurrent channels; then the existing soak ladder (4h -> 12h -> 24h) on the native runtime.
6. Package; repeat on two independently pristine Windows machines (the rc13 and Phase D lessons: "clean" must be provably clean).

**Runtime packaging:** minimal, checksum-locked GStreamer plugin closure derived from the classified required, conditional, and absence-tolerant manifests in CivicCast PR #283 (the verification doc is the seed) - not the 4.3 GB full runtime used for verification and not the broad GPL-containing `gstreamer-bundle` spike install. Packaging closure must account explicitly for the Windows guard/remap of `vah264enc`/`vah265enc` and must not promote absence-tolerant decoders into required dependencies. Licensed BOM per component: x264 present in the official runtime but the no-ship GPL decision stands; OpenH264, libav/codec patents, every bundled plugin; CVE/update policy named. Honesty note for release comms: download size may grow versus the 244 MB WSL-era installer (native bundles inline what WSL apt-installed); installed footprint and failure surface shrink.

**Session-0 proof obligations** (each its own check): DeckLink driver access from a service session, GPU encoder availability, service-account permissions, firewall rules, credential storage, **authenticated and authorized** Tauri-to-service IPC, and network-share/UNC media access under the service identity.

**Installer lifecycle proof obligations:** fresh install, upgrade, repair, uninstall, reboot, logout, and rollback behavior, with state removal or preservation matching the documented contract.

**Migration (right-sized for a beta-scale install base - the LPM program):** backup -> native install -> restore, with the WSL line as named rollback. No generalized in-place migration framework. Inventory beyond the database: media, configuration, certificates/secrets (incl. local CA keys), NATS state, Windows path conversion, service identity, runtime ownership transfer, tested rollback. (Database roles/permissions and extensions travel with the restore contract, section 5.) Keep the WSL line available until the native path reaches equivalent operational proof.

## 8. Program sequence (rolls up sections 3-7)

| # | Item | Gate to advance |
|---|---|---|
| 0 | Bootstrap Phase 0: audit-control genesis + CivicCast PR #283 | Both halves manually cross-reviewed; Scott approves; neither agent self-certifies |
| 1 | Release-truth manifest + extended live checker + real triggers (both repos) | Checker green against live GitHub; README generated from manifest |
| 2 | Postgres restore drill vs. shipping WSL product | Introspected source/restored equivalence proof, commit-bound artifact |
| 3 | Superseding ADR + claims-evidence rule + migration/cutover contract | ADR merged; claims verifier rejects a seeded false "implemented" claim (negative control) |
| 4 | Dual-runtime exclusion guard (section 6) | Guard implemented and proven **before any WSL machine receives a side-by-side native install**; bidirectional refusal demonstrated |
| 5 | Native supervisor + media-worker decision gate under session 0 | Gate evidence selects Python or Rust; all section 7 slice steps green on two pristine machines |
| 6 | Hardware + live-peer + multichannel + soak proofs | DeckLink physical, NDI relay, live multicast/unicast UDP, SRT/RTMP/RTSP peers, 3-channel, 24h soak |
| 7 | LPM cutover + rollback rehearsal | Cutover and rollback both executed and evidenced, incl. the post-activation data boundary (section 6) |

macOS is a **separate platform program** - the element inventory re-runs against the official macOS runtime in about an hour, but packaging, signing/notarization, launchd, and hardware validation are their own charter. Do not fold it into this one.

## 9. CivicSuite companion punch list

1. Adopt the WS-1 manifest + generated-README pattern. Current drift on record: 1.0.4 is Latest with a placeholder body ("promote from prerelease after clean-machine validation"); README's signing narrative was false for 1.0.2 (its release page: "not yet code-signed") and names a different signer path (SignPath) than what landed.
2. Record verified signing state: **1.0.4 MSI is validly Authenticode-signed** (verified 2026-07-16: SHA-256 matches published evidence `4d86e7b2...18ed42`; `Get-AuthenticodeSignature` -> Valid; CN=Scott Converse; issuer Microsoft ID Verified CS AOC CA 04 = Azure Trusted Signing; short-lived cert NotAfter 2026-07-14 remains valid via RFC-3161 timestamp - expected behavior, not a defect). Settle the 1.0.3 question once, in the manifest.
3. Close the July 9 Phase D findings trail (missing `msvcp140.dll` for bundled Python; Accessibility tab unreachable) with a genuinely pristine validation of the exact current MSI, per the program's own definition of done.
4. Treat macOS as a separate program here too (ADR-0008 is explicitly Windows-only).

## 10. Evidence appendix (what this charter's claims rest on, all verified 2026-07-16)

| Claim | Evidence |
|---|---|
| 51/51 required factories in official Windows runtime | CivicCast PR #283 at `e45b92d9b68c13d5ab5cd8a08caeeb9ce0e15301`, `.agent-runs/native-windows/spike-gstreamer-bundle/evidence/`: classified manifests and fresh-venv `Gst.ElementFactory.find` sweep; the earlier installer report is `C:\Users\scott\Desktop\CODE\civiccast-windows-native-gstreamer-verification.md` |
| Conditional and absence-tolerant inventory | Same PR #283 evidence: conditional 3/5 (`vah264enc`/`vah265enc` absent and Linux-only; Windows guard/remap open), absence-tolerant 5/11 (rank-demoted hardware decoders; absence harmless by design) |
| Native CPython/PyGObject spike | Same PR #283 evidence: native CPython 3.12, production-shaped caption pipeline to EOS, verbatim decode-back; source switching/recovery/multichannel/session-0 remain open |
| Caption pipeline native on Windows | Live pipeline -> 1,906,508-byte TS; 60 frames with A53 side data; ffmpeg-subcc decode-back returned injected text verbatim |
| rc13 withdrawn | Release page CAUTION block, `v1.0.0-rc13` |
| ADR-0003 rejected native Windows as "marginal" | `docs/adr/0003-project-hardware.md`, Option B |
| Runtime host is a WSL keeper | `main.rs` (`sleep infinity` keepalive; `systemctl restart` recovery) |
| NDI/SDI relays are BYO-ffmpeg | `civiccast/egress/ndi_relay.py`, `automation.py` relay markers |
| Executed DR drill was SQLite; no pg_restore | `docs/releases/0.5.0-dr-drill-verification.md`; `cli.py:2267`; repo-wide grep: one docstring hit |
| Live release checker exists (extension, not new) | `scripts/policy/check_published_release_assets.py` (isDraft/isPrerelease/tag/assets via `gh release view`) |
| Workflow triggers lack release-edit events | `.github/workflows/release-artifacts.yml` (`workflow_dispatch` + `push: tags: v*` only) |
| Roadmap verifier checks existence, not correctness | `scripts/roadmap_status.py` (`file_exists`, `migration_exists`) |
| CivicSuite 1.0.4 signed; 1.0.2 was not | MSI hash + `Get-AuthenticodeSignature` (Valid, Azure Trusted Signing); 1.0.2 release page ("not yet code-signed") |
| Phase D clean-machine failures | `docs/evidence/civicaccess-v102-phaseD-a11y-2026-07-09/README.md` |

**Owner decisions this charter does not make:** tagging/shipping anything, the cutover date for LPM, the macOS program start, and any NATS replacement. It drives to ready; those calls are yours.

---

## 11. Program roles (decided 2026-07-16, both reviewers concurring)

| Role | Assignee | Mandate |
|---|---|---|
| **Coder** | Claude | Implement the charter: code, tests, proof artifacts, documentation. Every completion claim ships with commit-bound evidence per section 4. Never self-certifies. |
| **Auditor** | Codex | Review every workstream against the charter, inspect actual code and runtime evidence, re-run falsifications (not just read diffs), reject unsupported completion claims, control advancement through the section 8 gates. |
| **Owner** | Scott Converse | Tags, releases, LPM cutover date, macOS program start, NATS replacement, merge/cutover authority, and tie-break on coder/auditor disagreement. |

Standing rules: the coder's known failure mode is optimistic overclaim - the gates exist to catch it; the auditor must not audit from reading alone - runtime evidence gets re-executed or attestation-verified; rung-3/4 items (superseding ADR, cutover, any tag) get dual review with both reviewers passing; neither model certifies its own work. The authoritative charter, gate, protocol, drift catalog, and verdicts live in `scottconverse/civiccast-audit-control`; CivicCast carries pointers and implementation evidence, not editable governance copies.
