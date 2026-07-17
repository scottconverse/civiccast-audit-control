# Native-Windows execution-spec design review

**Review date:** 2026-07-17

**Reviewer:** Codex, native-Windows program auditor

**CivicCast subject:** `8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a` on `program/native-windows`

**Review type:** pre-implementation design review; not a slice audit, not a verdict, and no gate advancement

**Execution posture:** `sandbox=danger-full-access`, `approval_policy=never`

**Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-specs-design-review-8fd3fa2f` (clean; ignored `.venv` created for one adversarial check)

## Executive conclusion

Do not implement these specifications unchanged. The architectural direction remains supportable, and the documents contain useful execution detail, but seven functional design defects would become blocking audit findings if built as written. The largest are:

1. the dual-runtime guard cannot reach its own native-active state and does not provide the charter-required bidirectional exclusion;
2. the supervisor's control and process-ownership contracts do not enforce their claimed security or single-owner properties;
3. the proposed native installer is not isolated from the existing WSL product and its forward-only database migration cannot safely auto-roll back by flipping binaries;
4. the claims verifier admits stale/wrong-SHA evidence and cannot detect unregistered claims; and
5. the migration proof demonstrably misses some corrupted media and does not define a coherent source snapshot.

The owner severity rule was applied: the findings below concern behavior, security, data integrity, or proof soundness. Wording-only issues are batched in SDR-011 and do not independently hold implementation.

## Finding summary

| ID | Severity | Affected documents | Design consequence |
|---|---|---|---|
| SDR-001 | Critical | dual-runtime guard, supervisor, ADR | Native activation is impossible with the rollback distro retained; deferring the WSL half permits double transmission. |
| SDR-002 | Critical | supervisor | Any interactive user receives mutating supervisor access, while child escape on supervisor failure can defeat single ownership. |
| SDR-003 | Critical | supervisor, playbook | The current production worker ownership/control path is Linux-FIFO-based and directly spawned by the egress daemon; the spec does not replace that contract. |
| SDR-004 | Critical | installer, dual-runtime guard, ADR | A single current-user installer identity and WSL-destructive hooks cannot safely produce a side-by-side per-machine native product. |
| SDR-005 | Critical | installer | Binary-only auto-rollback after a forward schema migration is not safe without mechanically proved compatibility or database restore. |
| SDR-006 | Critical | claims-evidence rule, README, playbook, ADR | Ancestor evidence, an unbound “most recent” JUnit, and an opt-in registry allow stale or omitted claims to pass. |
| SDR-007 | Critical | migration contract | The migration lacks a coherent quiesced snapshot, uses a demonstrably incomplete media comparison, and trusts stale step evidence on resume. |
| SDR-008 | Major | packaging closure, installer | PE imports and factory presence do not establish complete runtime closure, artifact authenticity, license clearance, or usable encoder fallback. |
| SDR-009 | Major | supervisor, installer, migration, README | Acceptance criteria omit charter-required service-identity surfaces and the full two-pristine-environment lifecycle proof. |
| SDR-010 | Major | README, packaging, migration, playbook, ADR | Several decisions are stated as settled before the owner has accepted the ADR, security risk, licensing posture, or NATS/data-loss posture. |
| SDR-011 | Minor, batched | all prose surfaces | A small set of path, authority, and overbroad factual phrasings should be reconciled before the near-release gate. |

## Functional findings

### SDR-001 — the dual-runtime guard is internally impossible and not bidirectional (Critical)

The guard says any registered CivicCast WSL distro is a positive refusal probe even when `ActiveRuntime=native`, but cutover deliberately leaves that distro installed for rollback. Therefore the documented successful cutover state immediately fails native preflight. The same section says ambiguous probes both “fail-open-to-refusal” and permit `ambiguous-start`, so the decision matrix has two answers for one state.

More importantly, native checks only at service start and the WSL-side refusal is explicitly allowed to remain unshipped. A keeper or WSL service can start after native preflight. That violates the charter and owner parallel-ship decision, both of which require bidirectional refusal before either product is installed side by side.

Evidence: [guard D2-D4 and deferred D3](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L14-L35), [cutover retains the distro](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L36-L51), [charter bidirectional gate](../CHARTER.md#L86-L93), and [owner parallel-ship decision](../decisions/2026-07-17-parallel-ship.md#L20-L29).

Acceptance to close:

- distinguish installed WSL rollback media from an active/healthy WSL owner;
- define one unambiguous failure policy for every selector/probe state;
- make the WSL refusal an installed-and-executed prerequisite to any co-install, not a future handoff;
- close the probe/start race with a continuously enforced or atomic ownership mechanism; and
- prove leftover distro, WSL restart after native start, concurrent cutover, partial cutover failure, unloaded-profile logon, and both service-start orders cannot produce two transmitters.

### SDR-002 — supervisor authorization and process ownership are not enforced (Critical)

The proposed pipe grants read/write to `INTERACTIVE` while exposing `start`, `stop`, `restart`, `drain`, and `runtime set`. Windows identifying the caller is authentication; granting every interactive token the same mutating command set is not the authorization promised by the charter. The only negative criterion checks a non-interactive token and therefore misses the allowed unprivileged interactive caller.

The “one process Windows owns” claim also lacks a Windows Job Object or equivalent kill-on-owner-close contract. If the supervisor crashes, is upgraded, or is forcibly stopped, children or descendants can survive and a replacement supervisor can start duplicates. The tests kill children but never kill the supervisor. `taskkill /pid` is described as a graceful NATS/worker signal without a proof that it flushes JetStream or tears down the worker cleanly.

Evidence: [supervisor D2-D7](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-supervisor.md#L13-L49), [guard-on-start only](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-supervisor.md#L54-L57), and [current acceptance criteria](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-supervisor.md#L76-L86).

Acceptance to close:

- separate read-only status access from administrator-only mutations and authorize the impersonated pipe client per command;
- specify explicit ACLs for executable, configuration, secret, log, database, and control surfaces under the LocalSystem threat model;
- contain every child and descendant in a kill-on-close Job Object (or demonstrate an equivalent property), with singleton startup;
- define and prove graceful shutdown for PostgreSQL, NATS/JetStream, control plane, and worker; and
- add negative controls for unprivileged interactive mutation, pipe squatting, malformed/oversized frames, supervisor crash during playout, orphan descendants, and concurrent stop/restart/cutover commands.

### SDR-003 — the native worker ownership and control contract is missing (Critical)

The spec makes the control plane and media workers sibling children of the Windows supervisor and calls the existing `worker.py <graph.json>` contract unchanged. In the current product, however, `GstPlayoutStrategy.start()` writes a graph and directly spawns the worker, returns its process handle to the egress daemon, and controls hot swap/reload/captions over a POSIX FIFO. The worker explicitly raises on Windows when FIFO support is unavailable. The decision-gate and session-0 smoke modes did not prove this production dynamic-control path.

The supervisor pipe's generic `start <child>` command does not define how schedule changes, graph/secret files, reloads, captions, per-channel process handles, or result events cross the new ownership boundary. Implementing only the listed build steps would either retain double ownership or lose live control.

Evidence: [supervisor process model and build steps](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-supervisor.md#L13-L20), [current strategy direct spawn](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/civiccast/egress/gst/strategy.py#L134-L158), and [worker's WSL/Linux-only FIFO](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/civiccast/egress/gst/worker.py#L28-L49).

Acceptance to close:

- specify the versioned control-plane-to-supervisor and supervisor-to-worker contracts, including request IDs, acknowledgements, errors, ownership, reconnect, and restart replay;
- replace or adapt the POSIX FIFO for Windows without weakening secret-file protections;
- trace and change every current process owner/caller, not only add a supervisor package; and
- prove schedule-driven start, hot reload, role swap, caption cue, worker crash/replay, control-plane restart, and supervisor restart for multiple channels.

### SDR-004 — the native and WSL installer products are not isolated (Critical)

The owner decided that WSL and native are two simultaneously offered products. The installer spec says “one installer source, two modes” but defines no separate product identifier, uninstall registration, install root, executable name, shortcut, update channel, or hook set. The current Tauri bundle is one `currentUser` product (`org.civiccast.installer`), while native must write Program Files/ProgramData and register a LocalSystem service. Its current NSIS hooks also delete the WSL autostart entry, terminate the distro on every uninstall, and unregister it on purge. A mode switch inside the application cannot make those bundle-time identities and uninstall hooks safe by itself.

Evidence: [installer D1-D4](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L7-L30), [current bundle identity and current-user mode](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/civiccast/apps/installer/src-tauri/tauri.conf.json#L1-L32), and [current WSL-destructive uninstall hooks](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/civiccast/apps/installer/src-tauri/nsis-hooks.nsh#L66-L100).

Acceptance to close:

- define distinct build-time Windows identities and hook sets for WSL and native, with an explicit elevation/per-machine boundary for native;
- install both artifacts in either order and show two independent Apps & Features entries, shortcuts, update/repair paths, state roots, and uninstallers;
- repair/uninstall/purge either product and prove the other's files, registry entries, runtime, data, and activation selector are unchanged; and
- add failure tests for denied UAC, partial service registration, existing same-version products, and product-code/update-channel collision.

### SDR-005 — upgrade auto-rollback is unsafe after schema migration (Critical)

The upgrade sequence flips to the new binary, runs `alembic upgrade head`, and promises automatic rollback by flipping the junction back. It explicitly does not downgrade the schema and relies on a policy that beta releases remain backward compatible. That policy is neither encoded nor checked before migration. A failed health gate can therefore restart an old binary against a schema it cannot safely read or write.

Evidence: [installer D3](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L18-L26) and [failed-upgrade proof row](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L41-L51).

Acceptance to close:

- either restore a verified pre-upgrade database backup on rollback or mechanically prove old-binary/new-schema compatibility before applying the migration;
- specify behavior for migration failure, power loss at every phase boundary, and rollback failure itself;
- test an actually incompatible migration as the negative control, not only a stubbed readiness failure; and
- bind the installed version, schema revision, backup, and rollback journal so recovery is resumable and idempotent.

### SDR-006 — the claims-evidence gate admits stale and missing proof (Critical)

An evidence SHA that is merely an ancestor of HEAD is not commit-bound to the claim being checked; code, tests, prose, or evidence inputs can change afterward. “Most recent CI junit artifact” is also undefined unless its repository, workflow, run attempt, source SHA, matrix completeness, and immutable provenance are checked. A JUnit from another SHA can satisfy the proposed rule.

The registry is opt-in and the verifier deliberately does not scan for unregistered claim statements. “A claim not in the registry has no standing” is a policy assertion, not enforcement: omitted claims remain visible to readers and invisible to the gate. Negative controls are recorded as commands but most need not have execution evidence.

Evidence: [claims rule D1-D3](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L9-L30) and [charter's full commit-bound evidence contract](../CHARTER.md#L61-L73).

Acceptance to close:

- bind every claim to the exact source SHA and relevant file blob IDs, or use an explicitly versioned validity range whose unchanged inputs are mechanically verified;
- validate JUnit provenance against the same GitHub repository, trusted workflow, run attempt, exact source SHA, expected test matrix, and passed node ID;
- make claim registration enforceable, for example through mandatory claim IDs/markers on strong prose plus a checker that rejects unregistered markers;
- require executed, source-bound negative-control results for the confidence class being claimed; and
- add adversarial tests for ancestor evidence after code mutation, JUnit from another SHA/run, skipped/xfail nodes, duplicate IDs, omitted claims, malformed registries, stale anchors, and evidence/test mutation after the CI run.

### SDR-007 — migration does not prove a coherent or complete transfer (Critical)

Database backup, media mirror, config copy, restore, and path rewrite occur without a station-wide write freeze or a final delta/reconciliation phase. The result can combine database state from one time with media/config from another. `robocopy /MIR` can also delete destination-only files without a preflight contract.

The media proof reuses `build_media_manifest`, which intentionally hashes only the head and tail of each file. I executed a same-size middle-byte mutation against the function at the reviewed SHA: the before and after fingerprints were identical and `corruption_detected=False`. Thus AC4 can pass while copied content is corrupt. Finally, `--resume` trusts completed-step evidence without binding that evidence to source identity, destination identity, tool version, inputs, or postconditions.

Evidence: [migration D1-D5](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-migration-contract.md#L9-L41), [resume contract](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-migration-contract.md#L47-L54), [acceptance criteria](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-migration-contract.md#L56-L65), and [bounded fingerprint implementation](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/civiccast/dr/backup.py#L197-L234).

Acceptance to close:

- define a bounded global quiesce/freeze, verified drain, database/media/config capture order, and final no-write or delta check before ownership transfer;
- use full-file cryptographic hashes for the one-time migration, or a separately justified copy-verification primitive that detects arbitrary content changes;
- make mirror deletion an explicit, previewed, operator-accepted action and test destination-only files;
- bind resume records to run ID, source/destination identities, source manifests, code/tool blobs, step inputs, and rechecked postconditions; and
- falsify concurrent source writes, middle-file corruption, interrupted copy/restore/path rewrite, stale evidence reuse, disk-full, path collision, secret ACL inheritance, and pending durable NATS messages.

### SDR-008 — packaging closure and trust are incomplete (Major)

Static PE-import closure does not include runtime-loaded DLLs, helpers, GI typelibs/schemas, codecs loaded by name, PostgreSQL share/extension data, Python native modules, or other non-PE resources. Exercising factories plus one caption and swap flow cannot prove all packaged application paths. The source artifact itself is called “official” but the spec does not require a pinned acquisition hash/signature before deriving the tree. At install time, a checksum file shipped beside a payload proves corruption only if the checksum is independently authenticated.

Factory presence is also insufficient for the encoder remap: `mfh264enc`/NVENC may exist but fail on the actual adapter, caps, profile, or driver. The doctor must run a representative encode, not cache only `ElementFactory.find`. Finally, `gst-inspect` metadata alone does not clear per-file/transitive licensing; OpenH264 binary distribution/patent posture and the exact FFmpeg build configuration require artifact-specific review and owner acceptance.

Evidence: [packaging D1-D6](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-packaging-closure.md#L8-L55) and [current acceptance criteria](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-packaging-closure.md#L67-L75).

Acceptance to close:

- pin and authenticate every upstream input, then derive closure in a clean isolated environment;
- add component-specific dynamic/resource closure and representative application-path execution, not only PE imports/factory discovery;
- authenticate the installed manifest through the signed installer or a separately signed manifest;
- require a real short encode/decode probe for selected hardware fallback; and
- produce file-level provenance/license mapping, exact FFmpeg configuration, required notices, and an owner-accepted OpenH264/legal posture.

### SDR-009 — the proof ladder omits charter obligations (Major)

The supervisor criteria omit explicit firewall, credential-storage, network-share/UNC, and service-account permission proofs required by charter section 7. They also do not prove dependency semantics: TCP accept is not NATS/JetStream readiness, and “dependents survive or re-ready” is not a measurable state transition.

The installer says the full matrix runs in Sandbox but only fresh install repeats on the second machine. Charter gate 5 requires all section-7 slice steps on two independently pristine environments. Under the owner testing decision, clean VMs/Sandbox can satisfy class 6; the owner's ordinary bare-metal development box is not pristine merely by being bare metal.

Evidence: [charter session-0 and lifecycle obligations](../CHARTER.md#L95-L112), [charter gate 5](../CHARTER.md#L114-L125), [supervisor criteria](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-supervisor.md#L76-L86), [installer venue and matrix](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L37-L51), and [owner testing calibration](../decisions/2026-07-17-testing-policy.md#L5-L17).

Acceptance to close:

- add independently observable criteria for authenticated NATS/JetStream readiness, database/NATS loss and recovery, firewall, secret storage, service-account ACLs, and UNC access;
- define exact ready/degraded/blocked transitions and dependency behavior with deadlines;
- run the complete lifecycle/runtime matrix on two demonstrably pristine class-6 environments, while leaving hardware/live-peer proofs at LPM and soaks capped at four hours; and
- capture clean-state provenance for each environment rather than inferring cleanliness from machine type.

### SDR-010 — owner decisions are presented as settled too early (Major)

The README and every spec label all decisions “made; do not reopen,” and the playbook directs future sessions to treat them as settled. That is unsafe before this design review and, for ADR-carried choices, before owner merge. In particular:

- accepting LocalSystem for beta and deferring least privilege is an explicit security-risk acceptance for the owner;
- shipping the exact OpenH264 binary/patent posture requires artifact-specific owner/legal acceptance;
- declaring JetStream state disposable must follow the stream-catalog proof and owner acceptance, not precede it;
- the ADR's opening “retires WSL2” conflicts with the owner's “neither replaces the other by decree” decision; retirement remains a future owner call on adoption and field evidence; and
- tags, releases, signing use, LPM cutover, and ADR-0003 supersession remain owner actions (the documents correctly preserve most of these boundaries).

Evidence: [README decision-complete claim](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/README.md#L3-L8), [playbook settled-decision instruction](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/.agent-runs/native-windows/specs/POST-FABLE-PLAYBOOK.md#L8-L17), [ADR opening decision](https://github.com/scottconverse/civiccast/blob/8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a/docs/adr/0021-native-windows-runtime.md#L12-L20), [owner parallel-ship language](../decisions/2026-07-17-parallel-ship.md#L6-L18), and [charter owner boundaries](../CHARTER.md#L155-L165).

Acceptance to close: mark specifications as proposed until this review's substantive items are reconciled; identify owner-risk decisions individually; record the owner's accept/reject decision; and preserve “sunset on evidence” as a future owner decision rather than an architectural fait accompli.

## Per-document review

### `.agent-runs/native-windows/specs/README.md`

1. **Design flaws:** “decision-complete” hides the unresolved guard, installer identity, proof-authority, and migration-integrity contracts (SDR-001, SDR-004, SDR-006, SDR-007).
2. **Charter conflicts:** implementation order is reasonable, but gate 3 cannot advance on the current claims rule and gate 4 cannot advance without shipped bidirectional refusal.
3. **Unverifiable criteria:** “CI green” is not enough unless the required proofs are enumerated and SHA/run-bound.
4. **Missing falsifications:** add a cross-spec conformance check covering owner decisions, gate prerequisites, exact test/evidence bindings, and prohibited `main`/tag/release actions.
5. **Owner boundary:** replace “every architectural decision is made HERE” with proposed/approved status per decision; owner-only items remain owner-only.

### `spec-supervisor.md`

1. **Design flaws:** insecure mutating IPC, no child-tree containment, no production worker-control seam, weak readiness probes, and undefined dependency transitions (SDR-002, SDR-003).
2. **Charter conflicts:** it claims authenticated/authorized IPC without authorization and omits several section-7 service-identity proofs (SDR-009).
3. **Unverifiable criteria:** “dependents survive or re-ready,” “existing alerting,” and output progress “where configured” lack observable states and deadlines.
4. **Missing falsifications:** unprivileged interactive mutation, supervisor death/orphans, pipe squatting/framing abuse, dynamic channel control, NATS graceful flush, dependency cascades, UNC, firewall, and secret ACLs.
5. **Owner boundary:** LocalSystem risk deferral needs explicit owner acceptance in ADR-0021; the auditor does not accept it implicitly through implementation.

### `spec-dual-runtime-guard.md`

1. **Design flaws:** impossible retained-distro state, contradictory ambiguity policy, startup-only TOCTOU, partial cutover/rollback, and incomplete profile restoration (SDR-001).
2. **Charter conflicts:** deferring the WSL refusal directly contradicts gate 4's bidirectional prerequisite.
3. **Unverifiable criteria:** fake process/Run-key fixtures cannot by themselves prove the real WSL service, keeper, registry interop, and both runtime start orders.
4. **Missing falsifications:** retained stopped distro, keeper starts after native, native starts after WSL, simultaneous cutovers, probe timeout/failure, unloaded user later logging in, each partial-step failure, and selector tampering.
5. **Owner boundary:** the rc-line patch landing is the owner/rc-line coder's action; native implementation must wait for that prerequisite rather than waive it.

### `spec-packaging-closure.md`

1. **Design flaws:** incomplete dynamic/resource closure, unauthenticated upstream/manifest trust, and factory-only encoder selection (SDR-008).
2. **Charter conflicts:** the minimal/no-GPL direction aligns, but the promised full component licensing/CVE closure is not met by `gst-inspect` plus PE imports.
3. **Unverifiable criteria:** two identical outputs prove determinism only for the same potentially contaminated inputs; “zero GPL” depends on complete and authoritative per-file classifications.
4. **Missing falsifications:** altered upstream artifact, poisoned PATH/registry, dynamic DLL/resource removal, nonfunctional present encoder, unexpected/orphan file, unknown license, and clean offline rebuild.
5. **Owner boundary:** GPL feature tradeoffs and exact OpenH264/codec patent-distribution posture require owner acceptance after evidence.

### `spec-installer-lifecycle.md`

1. **Design flaws:** no distinct Windows product/elevation identity, WSL-destructive inherited hooks, unauthenticated checksum trust, and unsafe schema rollback (SDR-004, SDR-005).
2. **Charter conflicts:** a native uninstall can touch WSL under the current reused hooks; only one lifecycle row repeats on the second environment (SDR-009).
3. **Unverifiable criteria:** “data intact,” “service healthy,” and “everything gone” need exact state inventories and product-specific boundaries.
4. **Missing falsifications:** install order both ways, uninstall/repair isolation, UAC denial, payload+checksum tamper, migration/power failure at every phase, locked files, junction ACL/tamper, incompatible schema, rollback failure, and cancel-before-purge.
5. **Owner boundary:** signing credentials, release publication, and any destructive field rollout stay owner-controlled; a purge remains an explicit operator choice.

### `spec-migration-contract.md`

1. **Design flaws:** no coherent snapshot, sampled media integrity, dangerous mirror deletion, stale resume evidence, and premature NATS disposability (SDR-007, SDR-010).
2. **Charter conflicts:** it must depend on an accepted WS2 native re-execution and separately prove media/config/secrets/identity; neither proof inherits from the other.
3. **Unverifiable criteria:** “same schedule with same media” and “full equivalence” lack exact snapshot time, full-file content proof, and accepted destination inventory.
4. **Missing falsifications:** concurrent writes, middle-byte corruption, destination-only files, interrupted/stale resume, insufficient disk, long/colliding paths, secret ACL failure, pending durable NATS messages, and rollback after native writes.
5. **Owner boundary:** LPM cutover and acceptance of any post-activation data-loss or NATS-loss boundary remain owner decisions. The stream catalog must be proved before a discard decision is treated as settled.

### `spec-claims-evidence-rule.md`

1. **Design flaws:** stale ancestor evidence, unbound JUnit provenance, unenforced opt-in registry, and recorded-but-unexecuted negative controls (SDR-006).
2. **Charter conflicts:** these fall below the charter's exact source/command/environment/result/provenance contract.
3. **Unverifiable criteria:** “most recent CI” and “a claim not in the registry has no standing” cannot be established from the proposed inputs.
4. **Missing falsifications:** wrong-SHA JUnit/evidence, mutated code/test after proof, skipped/xfail, missing matrix shard, unregistered strong claim, duplicate IDs, malformed schema, stale anchor, and forged local artifact.
5. **Owner boundary:** weakening an overclaim is not necessarily owner-only; correcting it is required. A product-position or scope change discovered while correcting it goes to the owner.

### `POST-FABLE-PLAYBOOK.md`

1. **Design flaws:** it operationalizes proposed decisions as settled and would route future coders into the defects above (SDR-010).
2. **Charter conflicts:** “auditor keeps full gate authority” should retain Scott's merge/cutover/tie-break authority; the auditor supplies the independent gate half, not ownership.
3. **Unverifiable criteria:** “review EVERY diff hostile” and “CI green” need the explicit slice proof matrix, not ritual language.
4. **Missing falsifications:** stale/mismatched verdict cache, interrupted audit, failed push/CI rerun, branch-head movement, owner-decision conflict, and audit-control/status drift.
5. **Owner boundary:** the playbook correctly forbids main/tag/release/LPM actions, but must not let “settled spec” substitute for owner approval.

### `docs/adr/0021-native-windows-runtime.md`

1. **Design flaws:** it incorporates the broken dual-runtime and claims-evidence designs, treats LocalSystem as accepted, and overstates a no-fork/failure-surface conclusion before packaging/lifecycle proof.
2. **Charter conflicts:** “retires WSL2” is too strong under the later owner decision that neither product replaces the other by decree; eventual retirement is an owner call on evidence.
3. **Unverifiable criteria:** “no fork,” “installed footprint and failure surface shrink,” and production-ready bidirectional exclusion are not established by the cited spikes.
4. **Missing falsifications:** ADR acceptance should be conditioned on closing SDR-001 through SDR-010 and on evidence links that distinguish spike runtime from shippable closure.
5. **Owner boundary:** this is correctly marked Proposed and rung-3 dual review. This review is the auditor half; owner merge is the other half and is what would supersede ADR-0003.

## ADR-0021 auditor position

**I would not pass ADR-0021 as written at `8fd3fa2f`.**

The native-Windows direction, retained Python worker decision, session-0 result, console-not-keeper boundary, NATS continuity, BYO-ffmpeg posture, and owner-gated LPM boundary are supportable. The ADR should return after:

1. the dual-runtime design is internally reachable and genuinely bidirectional;
2. the claims-evidence mechanism is exact-SHA/provenance-bound and registration is enforceable;
3. the supervisor/installer trust and side-by-side product boundaries are explicit;
4. the owner has explicitly accepted or changed the LocalSystem and codec-distribution risks; and
5. “retires/no fork/failure surface shrink” is reconciled to the parallel-ship decision and actual proof.

Even after an auditor pass, the ADR does not take effect until Scott merges it. A coder/auditor disagreement remains held for Scott; this review does not advance charter gate 3.

## Batched wording/documentation items (SDR-011, Minor)

- Narrow “no fork” to the observed media-engine code path; native adapters, installer, supervisor, guard, and CI are real platform divergence.
- Replace “installed footprint and failure surface shrink” with a hypothesis until packaging and lifecycle measurements exist.
- Use repo-valid paths from the ADR; bare `specs/...` does not identify `.agent-runs/native-windows/specs/...`.
- Replace “auditor keeps full gate authority” with language that preserves owner merge/cutover/tie-break authority.
- Correct `taskkill`/“SIGTERM-equivalent” only after the selected Windows shutdown mechanism is proven.
- Mark “decisions made; do not reopen” per-document with `Proposed`, `Auditor-reviewed`, and `Owner-approved` states.

## Evidence and confidence boundary

Checks performed:

- fetched and resolved `origin/program/native-windows` to the exact reviewed SHA;
- created and kept a separate detached worktree; final `git status --short --branch` was clean;
- read the complete nine-document change and reconciled it with `CHARTER.md`, `STATUS.md`, the owner severity/testing/parallel-ship decisions, ADR-0003, the current Tauri/NSIS installer, and the current GStreamer strategy/worker control path;
- `git diff --check 8fd3fa2f^ 8fd3fa2f` passed;
- executed the bounded-media adversarial mutation at the reviewed SHA: same file size, middle byte changed, identical fingerprint, `corruption_detected=False`; and
- confirmed PR #292 is a distinct, open WS2 audit thread at `0d021f72`; no WS2 verdict or gate action was taken here.

Confidence achieved: static/source design review plus one developer-machine adversarial proof. No installer, supervisor, dual-runtime, clean-machine, hardware, live-peer, or LPM runtime proof was attempted because those implementations do not yet exist and this request is review-only.

## What is already strong

- The documents preserve `program/native-windows`, no-tag, no-main, and owner-gated release/LPM boundaries.
- The Python-vs-Rust decision now follows recorded evidence rather than preference.
- The packaging spec correctly rejects the broad spike bundle and requires deterministic minimal closure and a BOM.
- The installer explicitly covers fresh/upgrade/repair/uninstall/reboot/logout/rollback states rather than only happy-path install.
- The migration contract names database, media, config, secrets, NATS, paths, ownership, and rollback instead of claiming database restore is the whole migration.
- The playbook preserves exact-SHA audits, canonical verdict authority, and honest CI/proof language.

Those strengths are worth keeping while the functional contracts above are repaired.
