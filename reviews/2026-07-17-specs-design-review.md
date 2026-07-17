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

---

## Round 2 — closure assessment at `397dad2c`

**Assessment date:** 2026-07-17

**CivicCast subject:** `397dad2c579328aa09479692606231253c172435` on `program/native-windows`

**Compared with:** Round 1 subject `8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a`

**Review type:** exact acceptance-criterion closure review; not a slice audit, not a verdict, and no gate action

**Execution posture:** `sandbox=danger-full-access`, `approval_policy=never`

**Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-specs-design-review-r2-397dad2c` (clean)

### Round 2 conclusion

The revision is a substantial improvement, but it does **not** genuinely close all Round 1 acceptance criteria.

| Status | Count |
|---|---:|
| CLOSED | 2 |
| PARTIALLY CLOSED | 9 |
| NOT CLOSED | 0 |

Material gaps remain in dual-runtime exclusion, supervisor security, worker command delivery, cross-uninstall behavior, upgrade recovery, claims authority, migration freeze/inventory, and credential protection. `SDR-011` also retains several wording/path items, but those remain batched Minor work under the owner severity decision.

**The auditor half of ADR-0021's rung-3 review does not pass at `397dad2c`.** The owner half remains Scott's merge, but there is not yet an auditor pass for him to pair with it. In particular, ws3 claims-evidence implementation should not start against this v2 contract unchanged; the `SDR-006` items below alter the verifier's trust model and tests.

### Closure matrix

| Finding | Round 2 status | What the revision genuinely closes | What remains |
|---|---|---|---|
| SDR-001 | **PARTIALLY CLOSED** | Installed WSL is distinguished from active WSL; the decision table is singular; co-install is conditioned on an owner-landed WSL patch; cutover interruption and both keeper start orders are named. | The mutex is held by the Windows keeper, not the enabled in-distro systemd service. That service can start without the keeper, or outlive a crashed keeper. Native then relies on a 30-second poll, permitting a double-transmit window; a probe error explicitly starts native. The WSL service itself still lacks the required refusal. |
| SDR-002 | **PARTIALLY CLOSED** | Per-command impersonation/authorization, administrator-only mutations, Job Object containment, singleton intent, graceful-stop contracts, frame caps, command serialization, and supervisor-death tests are now specified. | The supervisor pipe has no explicit DACL/remote-access contract. `FILE_FLAG_FIRST_PIPE_INSTANCE` detects a pre-existing first instance but does not make pre-creation “squat-proof”; the legitimate service receives `ERROR_ACCESS_DENIED`. Recovery/availability and named-object security descriptors remain unspecified. |
| SDR-003 | **PARTIALLY CLOSED** | Worker ownership correctly stays with `GstPlayoutStrategy`; the Windows change is scoped to FIFO-to-named-pipe transport; real strategy-path multichannel/reload/swap/caption tests are required. | The original closure required request IDs, acknowledgements, errors, reconnect, and restart replay. V2 deliberately retains the one-way line protocol. A successful pipe write is not proof the worker applied the command, and broken-pipe retry does not resolve lost/duplicated commands. |
| SDR-004 | **PARTIALLY CLOSED** | Native now has a distinct per-machine identity, hooks, ARP entry, install root, update channel, and both-order coexistence/cross-uninstall matrix. | The new “selector untouched” uninstall rule can orphan ownership: uninstall native while `ActiveRuntime=native` and the remaining patched WSL product refuses; the reverse occurs for `wsl`. Reporting the orphan after uninstall does not preserve an operable station. |
| SDR-005 | **PARTIALLY CLOSED** | Rollback now restores a real WS2 backup; incompatible-schema, phase journal, power-loss, health-failure, and schema-failure controls are named. | The sequence takes the backup before stopping writers, and explicitly accepts losing writes between backup and failure. Stop/freeze must precede the recovery point and remain held. The journal also needs normative bindings to old/new version, schema revisions, verified backup identity/hash, and defined rollback-restore failure handling. |
| SDR-006 | **PARTIALLY CLOSED** | Strong-claim markers are enforceable in a governed set; ancestry is replaced by exact binding plus blob identities; JUnit repo/workflow/run/attempt/head fields and key falsifications are named. | The spec does not require the minimum bound input set, so a registry entry can omit its prose, code module, test, verifier/schema, workflow, or evidence generator. It does not prove expected-matrix completeness. Non-CI controls are supported by a self-reported `last_executed` mapping rather than a hashed/provenanced command artifact, and a missing current control may merely “degrade” while the claim still passes. |
| SDR-007 | **PARTIALLY CLOSED** | Full-file SHA-256 replaces sampling; `/MIR` is removed; destination must be empty; resume records gain identities/hashes/postconditions; NATS discard is owner-gated after inventory; interruption/disk/path/ACL controls are named. | “Stop ALL services” conflicts with needing PostgreSQL live for `pg_dump`, and the freeze has no durable interlock that prevents or records a transient service restart. A final current-process check can miss start-write-stop. Config/secrets inventory still trusts the bootstrap write list instead of the runtime read set plus operator-added/external configuration. |
| SDR-008 | **CLOSED** | The design now covers pinned publisher artifacts, clean derivation, static/dynamic/resource closure, signed-manifest trust, representative packaged-tree paths, real encoder probing, per-file provenance/license mapping, and an owner evidence memo for OpenH264/FFmpeg. | No Round 1 design acceptance item remains. Implementation must still verify publisher signatures/checksums and prove every named negative control; this closure is design-only. |
| SDR-009 | **PARTIALLY CLOSED** | Firewall, UNC, ACL audit, authenticated JetStream readiness, dependency transitions, two full pristine class-6 matrices, and clean-image provenance are explicit. | Credential AC-S2 presents machine-scope DPAPI **or** restrictive file ACL as interchangeable. Microsoft documents that any user on the same computer can decrypt `CRYPTPROTECT_LOCAL_MACHINE` data. A service-only or file ACL remains mandatory; machine scope alone cannot satisfy the non-admin denial criterion. |
| SDR-010 | **CLOSED** | Decision states, owner-acceptance register, LocalSystem/legal/NATS/downtime boundaries, parallel-ship posture, future owner sunset, ADR merge authority, and auditor-versus-owner authority are now explicit. | No material Round 1 acceptance item remains. Owner acceptance is still pending by design. |
| SDR-011 | **PARTIALLY CLOSED** | `taskkill`/“SIGTERM-equivalent” was replaced with per-child mechanisms, and authority/decision-state wording was corrected. | ADR-0021 still says “Zero porting cost; no fork” although the supervisor spec now requires worker/strategy Windows branches; still asserts footprint/failure surface shrink before measurement; and still uses nonexistent root paths `specs/...` instead of `.agent-runs/native-windows/specs/...`. These remain batched Minor unless used as decision evidence. |

### Detailed acceptance checks

#### SDR-001 — bidirectional exclusion

The retained-distro contradiction and the old ambiguity-table contradiction are fixed in [guard D2-D3](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L14-L36). The prerequisite language is also correctly promoted in [D6](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L49-L56).

The continuous/atomic acceptance item is not met. The current WSL installation enables `civiccast.service` with `Restart=always`; starting the distro by any route can start the service independently of the patched Windows keeper. The revision assigns the mutex only to that keeper. If the keeper dies while the systemd service continues, the mutex is abandoned and can be acquired by native even though WSL remains active. Microsoft also cautions that an abandoned mutex means the protected resource may be indeterminate; treating it as acquired-with-log is not sufficient without proving the old transmitter is gone ([Microsoft mutex documentation](https://learn.microsoft.com/en-us/windows/win32/sync/mutex-objects)).

Smallest closure: put the selector/ownership refusal in the WSL service startup path itself (for example, an owner-reviewed `ExecCondition`/`ExecStartPre` plus a lifetime mechanism tied to the transmitting service), fail before transmission when authority cannot be checked, and test direct distro/service starts, keeper crash while service stays alive, and probe failure—not only keeper start orders.

#### SDR-002 — supervisor security and ownership

Job containment, per-command authorization, explicit storage ACLs, graceful stops, and the major negative controls are present in [supervisor D3-D7](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-supervisor.md#L31-L80) and [AC-N1 through AC-N5](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-supervisor.md#L116-L126).

Two security details remain:

1. Define the actual supervisor-pipe security descriptor, including local-only intent/`NETWORK` denial and narrowly assigned create-instance rights. Microsoft notes that a default named-pipe descriptor grants read access to Everyone and anonymous, and that `FILE_GENERIC_WRITE` includes `FILE_CREATE_PIPE_INSTANCE` ([named-pipe security](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)).
2. Rename and repair AC-N2. `FILE_FLAG_FIRST_PIPE_INSTANCE` makes a later instance fail with `ERROR_ACCESS_DENIED`; it does not evict a rogue first server or keep the legitimate service available ([CreateNamedPipeW](https://learn.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-createnamedpipew)). Client server-identity verification prevents sending commands to the rogue endpoint, but the service still needs a specified fail-closed recovery/diagnostic path and a real pre-creation test.

Apply equally explicit security descriptors to the global singleton/runtime mutexes so an unprivileged local process cannot forge ownership or hold the station offline.

#### SDR-003 — worker command transport

Preserving egress-daemon ownership is the right correction. It removes the Round 1 double-owner problem and correctly scopes the platform seam in [supervisor D2](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-supervisor.md#L12-L30).

The exact command-delivery acceptance item remains. The current strategy returns success after `os.write`; the worker logs application/failure later with no acknowledgement path. V2 says the line protocol remains unchanged, so it still cannot distinguish “written,” “parsed,” and “applied.” Specify a versioned envelope with command ID and result acknowledgement, bounded retry/idempotency rules, reconnect behavior, and replay policy for reload/swap/caption/stop—or explicitly narrow the product guarantee and obtain owner acceptance for command loss. Then mutation-test lost response, duplicate delivery, worker restart between write/apply, and reconnect with multiple channels.

#### SDR-004 — side-by-side installer identity

The distinct product identity and both-order lifecycle matrix in [installer D1](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L10-L23) close the original bundle-identity and inherited-hook defects.

The new selector-orphan behavior is functional. Required state table:

| Before uninstall | Action | V2 resulting state |
|---|---|---|
| both installed, `ActiveRuntime=native` | uninstall native | WSL remains but its patched guard refuses native ownership |
| both installed, `ActiveRuntime=wsl` | uninstall WSL | native remains but D3 refuses while selector is `wsl` |

Smallest closure: block removal of the active product until an explicit, successful cutover/rollback transfers ownership, or offer that transfer as a separately acknowledged transaction before uninstall. Prove the remaining product is operable—not merely byte-unchanged—after every cross-uninstall row.

#### SDR-005 — upgrade recovery

Database restore and incompatible-schema falsification are now real design elements in [installer D3](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L29-L43).

The recovery point is ordered incorrectly: backup occurs at step 2, service stop at step 3, and writes in between are knowingly discarded after failure. Stop/drain all writers first, prove quiescence, take and verify the backup, then keep the freeze through schema migration/health commit. The phase journal must bind old/new product versions, pre/post schema revisions, backup manifest/blob identity, verification result, and rollback outcome. Add an injected restore failure after the incompatible migration; “installer resumes” is not a defined recovery if the recovery operation itself fails.

#### SDR-006 — claims authority

The marker approach and wrong-SHA/skip/blob/malformed controls are meaningful improvements in [claims D1-D5](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L12-L50).

Before ws3 implementation, require these schema invariants:

- `inputs` must include, at minimum, the `where` file, code-defining module, test file, verifier and schema, evidence generator, trusted workflow definition, and any claim-specific fixtures/configuration;
- JUnit provenance must validate the expected job/matrix shards and collection floor, not only one passing node in one artifact;
- every required negative control must have a durable artifact containing exact command, environment/tool versions, timestamps, exit status, stdout/stderr hash, source SHA, input blob IDs, and provenance; editing `last_executed.result` is not proof; and
- a claim may exit 0 only at the confidence class its required current controls support. Missing current controls fail or produce a non-pass status consumed by the claims gate—not an informational degradation beside a green claim.

Add the Round 1 mutations that remain absent: omit the test/verifier/workflow from `inputs`, mutate the test after JUnit, remove a matrix shard, and forge `last_executed` without an execution artifact.

#### SDR-007 — migration integrity

The revision correctly adopts full hashes, non-destructive destination handling, evidence-bound resume, and an owner-gated stream decision in [migration D1-D7](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-migration-contract.md#L12-L53).

The coherent-snapshot contract still needs an enforceable freeze:

- say explicitly that application publishers/workers stop while PostgreSQL stays up solely for the migration connection; “stop ALL CivicCast services” is otherwise executable as “stop the database before pg_dump”;
- install a maintenance/freeze interlock honored by every app/keeper/service start path, with a journaled generation/owner, rather than relying on a final current-process observation that misses a transient start-write-stop; and
- inventory config/secrets from actual runtime readers, service/unit environment files, operator overrides, and external paths. The installer's historical write list is a seed, not an authoritative read set.

Falsify a service that starts, writes, and exits before the final check; an operator-added config file outside the bootstrap list; source mutation during full hashing; and a resume after source/destination identity reuse with changed content.

#### SDR-008 — packaging closure

**CLOSED.** [Packaging D1-D6](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-packaging-closure.md#L11-L56) and [AC1-AC8](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-packaging-closure.md#L58-L74) cover each Round 1 design acceptance item. This does not pre-accept implementation evidence: the slice must verify actual publisher signatures/checksums, actual file-access traces, real encoder fallback, signed-manifest chaining, and the owner memo.

#### SDR-009 — proof ladder

The session-0 surfaces and complete two-environment matrix are now explicit in [supervisor AC-S1 through AC-S4](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-supervisor.md#L86-L98) and [installer D7](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L55-L61).

AC-S2's “DPAPI machine-scope (or file ACL)” alternative is false as a secrecy boundary. Microsoft states that with `CRYPTPROTECT_LOCAL_MACHINE`, any user on that computer can decrypt the data ([CryptProtectData](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)). Require the restrictive secret-file ACL in all cases, or choose a service-specific protection mechanism; then prove a non-admin who obtains the ciphertext still cannot decrypt it. The remote firewall request should be executed from a clean software-lab VM/Sandbox peer unless deferred to LPM, preserving the owner testing policy.

#### SDR-010 — owner authority

**CLOSED.** The [decision-state ladder and owner register](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/README.md#L8-L27), [playbook authority wording](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/.agent-runs/native-windows/specs/POST-FABLE-PLAYBOOK.md#L3-L17), and [ADR owner-risk register](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/docs/adr/0021-native-windows-runtime.md#L64-L77) preserve the owner boundary and parallel-ship decision.

#### SDR-011 — batched prose/path reconciliation

**PARTIALLY CLOSED, Minor/batched.** The process-stop and authority language is repaired. These exact Round 1 items remain:

- [ADR lines 28-34](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/docs/adr/0021-native-windows-runtime.md#L28-L34) still say “Zero porting cost; no fork,” while supervisor D2 requires Windows branches in `worker.py` and the strategy. Narrow this to “the media engine and graph model ran unmodified; native adapters remain owed.”
- [ADR lines 60-62](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/docs/adr/0021-native-windows-runtime.md#L60-L62) still assert footprint/failure-surface shrink before the promised measurement. Mark it a hypothesis.
- [ADR lines 81-92](https://github.com/scottconverse/civiccast/blob/397dad2c579328aa09479692606231253c172435/docs/adr/0021-native-windows-runtime.md#L81-L92) still cite root paths `specs/...`, which do not exist. The valid root is `.agent-runs/native-windows/specs/...`; the explanatory parenthetical does not make the literal path resolvable.

### Revision-introduced findings

These are included in the partial statuses above rather than assigned a new audit round or verdict:

1. **Active-product uninstall can orphan `ActiveRuntime`** (SDR-004; functional availability).
2. **Machine-scope DPAPI is offered as a standalone secrecy alternative** (SDR-009; security).
3. **`FILE_FLAG_FIRST_PIPE_INSTANCE` is labeled squat-proof although pre-creation denies the legitimate server** (SDR-002; local availability/security semantics).
4. **Upgrade recovery explicitly tolerates post-backup/pre-stop write loss** (SDR-005; data integrity).

### Round 2 ADR-0021 auditor position

**Auditor half: DOES NOT PASS at `397dad2c579328aa09479692606231253c172435`.**

The ADR's parallel-ship framing and owner-risk register are now correct, but it incorporates a guard that still lacks in-distro service refusal and a claims rule whose evidence inputs/negative controls remain forgeably incomplete. It also retains the “no fork” claim while its companion supervisor spec explicitly adds Windows worker/strategy branches. Those are not an adequate rung-3 record for owner merge.

Smallest route to an auditor pass:

1. close SDR-001's WSL-service lifetime/refusal path;
2. close SDR-006's mandatory input/provenance/control schema before ws3 implementation;
3. reconcile the ADR's remaining false/path claims;
4. repair the material SDR-002/003/004/005/007/009 contracts above; and
5. return the exact revised SHA for a scoped Round 3 design closure review.

### Round 2 evidence and boundary

- Resolved live `origin/program/native-windows` and the requested abbreviated SHA to exact commit `397dad2c579328aa09479692606231253c172435`; parent is exactly Round 1 subject `8fd3fa2fc9521e28cf4b0d3fc9897bea7a5d9c6a`.
- Created a separate detached worktree and confirmed it remained clean.
- Read the complete nine-document final state and the full `8fd3fa2f..397dad2c` diff; `git diff --check` passed.
- Rechecked current WSL runtime behavior: the Windows keeper owns a `wsl.exe ... sleep infinity` child and can restart `civiccast.service`; the installed unit is enabled with `Restart=always`, confirming the service lifetime is not identical to the proposed keeper-held mutex lifetime.
- Rechecked current worker command behavior: strategy success is an `os.write` result; worker application/failure is later log output with no acknowledgement channel.
- Verified ADR literal `specs/...` paths do not exist while the `.agent-runs/native-windows/specs/...` paths do.
- Consulted Microsoft primary documentation for named-pipe DACL/first-instance semantics, mutex abandonment, and machine-scope DPAPI.
- PR #292 remained a separate WS2 thread. No WS2 source, verdict, CI result, or gate state was audited or changed.

Confidence: static/source design review with adversarial state-machine counterexamples and primary-platform-contract verification. No implementation runtime was claimed because these remain specifications.

## Round 3 — closure assessment at `82416e05`

**Assessment date:** 2026-07-17

**CivicCast subject:** `82416e0577e4f13d67b1b9b430016a195b1c1aa9` on `program/native-windows`

**Compared with:** Round 2 subject `397dad2c579328aa09479692606231253c172435`

**Review type:** exact Round-2 acceptance-criterion closure review; not a slice audit, not a verdict, and no gate action

**Execution posture:** `sandbox=danger-full-access`, `approval_policy=never`

**Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-specs-r3-82416e05` (clean)

### Round 3 conclusion

V3 closes the Windows named-object security contract, the packaging contract, the owner-authority boundary, and the ADR's remaining false/path prose. It does **not** close every material Round-2 acceptance item.

| Status | Count |
|---|---:|
| CLOSED | 4 |
| PARTIALLY CLOSED | 7 |
| NOT CLOSED | 0 |

The remaining items are functional or proof-contract defects under the owner's severity calibration: a native guard probe error still authorizes transmission; worker restart replay is undefined for two command classes; installer and migration acceptance rows contradict their normative decisions; the upgrade freeze lacks an enforceable start-path/read-only-health contract; the claims design has an exact-SHA/run-ID self-reference loop as well as incomplete non-CI provenance and mandatory-input mechanics; and the firewall proof venue remains unbound to the owner's software-lab/LPM boundary.

**The auditor half of ADR-0021's rung-3 review still does not pass at `82416e05`.** The ADR incorporates the dual-runtime and claims contracts, and those retain material gaps. **WS3 implementation is not unblocked against the v3 claims spec:** `SDR-006` still changes the verifier schema and trust model. This design result is independent of WS2's separate audit/gate state.

### Closure matrix

| Finding | Round 3 status | What v3 genuinely closes | Exact remainder |
|---|---|---|---|
| SDR-001 | **PARTIALLY CLOSED** | WSL refusal now runs in the in-distro service's own start path, fails closed on unreadable host authority, is rechecked on service start/restart, and has direct-start plus keeper-crash controls. Abandoned mutex ownership no longer authorizes native without A2. | Native's decision table still says an A2 probe error/timeout starts transmission. That is the exact fail-open path Round 2 identified: a keeperless but still-running WSL service plus an unavailable A2 probe can overlap native. |
| SDR-002 | **CLOSED** | Explicit pipe and global-object security descriptors, local/NETWORK boundary, service-only create-instance right, first-instance detection semantics, degraded recovery, client server-identity check, and real pre-creation control are all specified. | No Round-2 design acceptance item remains. Implementation must realize “read/write” as individual pipe rights that do not accidentally regrant `FILE_CREATE_PIPE_INSTANCE`; the spec already identifies that constraint. |
| SDR-003 | **PARTIALLY CLOSED** | The Windows worker pipe now has a versioned request/result envelope, command IDs, applied/error acknowledgement, bounded idempotent redelivery, timeout surfacing, reconnect, and desired-state replay. | Restart replay names graph reload and role only. Caption cues and `stop` are real commands in `parse_control_line`, but their post-restart/lost-ack policy is unspecified. The Round-2 acceptance criterion explicitly required replay policy for reload/swap/caption/stop. |
| SDR-004 | **PARTIALLY CLOSED** | D1 blocks active-product removal until an acknowledged transfer, clears the selector for sole-product removal, preserves inactive-product ownership, and requires operability of the survivor. | The lifecycle proof matrix still says every cross-uninstall leaves the selector untouched. That is impossible for active-product transfer and sole-active removal, and gives the implementer two incompatible acceptance contracts. |
| SDR-005 | **PARTIALLY CLOSED** | Backup follows stop/drain, is verified before mutation, the journal binds the requested identities/outcomes, and rollback-restore failure halts stopped with recovery material preserved. | “The freeze holds” is asserted but no start-path interlock or maintenance/read-only health mode defines how it survives operator/SCM starts. Step 6 starts the service while the freeze is said to remain held, without saying how writes stay blocked. D3 also claims injected restore failure is a proof-matrix row, but the matrix contains no such row. |
| SDR-006 | **PARTIALLY CLOSED** | Mandatory input categories, exact blob/SHA binding, expected job/matrix completeness, collection floor, artifact-backed controls, fail-on-missing/stale behavior, and the Round-2 mutations are substantially present. | A committed registry cannot pre-record the CI `run_id`/artifact for its own final SHA; committing those results changes the SHA, so D2/D3/D7 form a self-reference loop. The flat `inputs` list also cannot mechanically infer an omitted claim-specific fixture/config, and non-CI artifacts have no trusted workflow, signer/attestation, or other immutable execution provenance. |
| SDR-007 | **PARTIALLY CLOSED** | PostgreSQL-versus-app freeze wording, the shared D7a maintenance record, runtime-read-set inventory, and generation/owner journaling close the three principal design gaps. | AC5 still expects a mid-migration start/write to be detected only by the final check, contradicting D1's requirement that the interlock prevent the start/write. The requested controls for operator-added config, mutation during full hashing, and reused identities with changed content are not enumerated; the stale-hash test is not a substitute for each failure mode. |
| SDR-008 | **CLOSED** | Unchanged from Round 2. | No Round-1/2 design acceptance item remains; implementation evidence is still owed by the slice. |
| SDR-009 | **PARTIALLY CLOSED** | Restrictive secret-file ACL is now mandatory; machine-scope DPAPI is correctly demoted to optional defense in depth. | AC-S1 still says only “a remote-device request.” It does not bind that request to a clean software-lab VM/Sandbox peer or explicitly defer it to LPM, as Round 2 required and the owner testing policy constrains. |
| SDR-010 | **CLOSED** | Unchanged from Round 2; owner decisions remain explicit and unsettled until owner acceptance. | No material acceptance item remains. |
| SDR-011 | **CLOSED** | The ADR now scopes the no-port result to the engine/platform seam, marks footprint/failure-surface reduction as an unasserted hypothesis pending measurement, and uses resolvable repo paths. | No Round-2 item remains. Residual “v2” section labels in v3 documents are batched wording only and do not change action. |

### Detailed acceptance checks

#### SDR-001 — bidirectional exclusion

The [in-service placement in guard D4](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L37-L56) and [new AC3b/AC3c controls](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L108-L129) directly fix the keeper-lifetime defect. The abandoned-mutex rule is also correctly conservative.

The closure is incomplete because [D3 still authorizes native start on any probe error or timeout](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-dual-runtime-guard.md#L25-L36). A2 is the lifetime proof when the keeper is absent or its mutex is abandoned. If A2 cannot be read while the WSL transmitter remains alive, “start + probe-degraded” permits the forbidden double-transmit state.

Smallest closure: make unreadable/timeout A2 non-authorizing before native transmission (a bounded blocked/retry state is acceptable), and add a control with a live keeperless WSL service plus an injected A2 timeout that proves native never starts.

#### SDR-002 — supervisor pipe and global objects

**CLOSED.** [Supervisor D7](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-supervisor.md#L82-L104) now states the principals, remote boundary, narrow create-instance authority, first-instance failure behavior, degraded recovery, and client identity verification. [AC-N2](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-supervisor.md#L143-L150) is a real pre-creation test. This matches Microsoft's distinction between named-pipe data rights and `FILE_CREATE_PIPE_INSTANCE`, and between first-instance detection and availability loss.

#### SDR-003 — acknowledged worker commands

The [new envelope](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-supervisor.md#L28-L41) is the right architecture: acknowledgement follows application, IDs make same-command redelivery idempotent, and timeout becomes a channel error.

The current parser includes `swap`, `reload`, `caption`, and `stop`, but v3's restart replay defines only current graph reload plus role. A caption is time-sensitive and cannot safely inherit generic “desired state”; a stop is terminal and cannot safely inherit graph/role replay. Smallest closure: state the at-most/exactly-once and reconnect policy for each of the four verbs, including whether an applied-but-unacknowledged caption is replayed and whether unacknowledged stop suppresses restart, then make the named lost-ack/restart controls parameterized across those policies.

#### SDR-004 — uninstall ownership

[Installer D1](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L10-L31) closes the orphaned-selector behavior. The remaining defect is the contradictory [Cross-uninstall matrix row](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L78-L93).

Smallest closure: split the row into inactive-product removal (selector unchanged), active product with survivor (successful acknowledged transfer, then removal), active sole product (selector cleared), and refused/cancelled transfer (nothing removed). Every successful survivor row must still prove actual start/transmit operability.

#### SDR-005 — upgrade recovery

[Installer D3](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-installer-lifecycle.md#L37-L58) now orders backup correctly and defines the rollback-stop boundary. It does not yet define the mechanism that makes the stated freeze true. A stopped service is not a durable interlock against an explicit start, and starting it for health at step 6 needs a specified maintenance/read-only mode if the freeze remains held.

Smallest closure: acquire the shared D7a interlock (or an equivalently defined upgrade interlock) before drain, require every native start path to honor it, run a read-only health gate while writers remain refused, and release only on commit. Add the promised injected rollback-restore-failure row plus a concurrent manual/SCM start attempt to the actual matrix.

#### SDR-006 — claims-evidence authority

[Claims D2-D5](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L23-L64) incorporates most of the Round-2 checklist. Three trust properties remain unresolved:

1. **The exact-SHA/same-run registry is self-referential.** D2 says the committed entry contains `source_sha`, `run_id`, `run_attempt`, and `head_sha`; D7 says it consumes JUnit from the same run. The run ID and artifact do not exist until CI runs the commit. Committing them afterward creates a new HEAD that no longer equals the artifact's `head_sha`. A committed control artifact that claims its own final containing SHA has the same fixed-point problem. AC1 therefore has no specified realizable steady state.
2. A schema can assert that a list contains declared paths, but it cannot infer an omitted claim-specific fixture/config unless the entry declares typed required roles or a trusted per-claim manifest defines them. The displayed registry shape has no such role mapping. D5 should mutate every mandatory class, including `where`, code, schema, generator, and fixture/config—not only test/verifier/workflow.
3. A file that reports its own command, timestamp, exit, and environment is durable but not immutable execution provenance. For CI-safe controls, the trusted run can supply provenance. For controls CI cannot rerun—including the first-scope session-0 claim—the charter permits attestation as integrity/provenance while retaining raw observations and review. V3 instead says “not attestations” and defines no replacement trust root.

Smallest closure: remove committed result metadata from the self-referential side of the contract. For example, let a committed claim definition bind expected source inputs and a trusted resolver, then let the verifier consume the current workflow's runtime run/attempt context or an external immutable evidence record keyed by the already-existing source SHA. Use typed required-input roles (with an explicit mechanism for claim-specific dependencies), and require each control artifact to chain either to the trusted CI run/attempt or to a signed/attested evidence envelope whose limitations remain explicit. Add self-reference, missing-category, and forged-untrusted-artifact controls.

#### SDR-007 — migration coherence

[Migration D1 and D4](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-migration-contract.md#L12-L51) now state the correct freeze and inventory sources. The acceptance section was not reconciled: [AC5](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-migration-contract.md#L71-L86) still treats a mid-migration service start/write as something the final comparison detects. Under D1, the start itself must be refused and no write may occur.

Smallest closure: rewrite AC5 to assert interlock refusal/no write, then separately corrupt/bypass the interlock to prove the final no-write check is defense in depth. Add explicit controls for operator-added config outside the bootstrap list, source mutation during full hashing, and resume after identity reuse with changed content.

#### SDR-008 — packaging closure

**CLOSED, unchanged.** The design remains complete; no implementation proof is pre-accepted.

#### SDR-009 — session-0 proof ladder

The [credential rule](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/.agent-runs/native-windows/specs/spec-supervisor.md#L110-L125) correctly treats the ACL as mandatory and machine-scope DPAPI as non-authoritative defense in depth. The firewall row still needs a venue consistent with the owner testing policy: clean VM/Sandbox peer for software-lab proof, or explicit LPM deferral for real-world proof. “Remote-device request” alone leaves the required confidence class and allowed venue ambiguous.

#### SDR-010 — owner authority

**CLOSED, unchanged.** The decision-state and owner-acceptance register remain intact.

#### SDR-011 — ADR accuracy and paths

**CLOSED.** [ADR-0021's falsified-premise text](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/docs/adr/0021-native-windows-runtime.md#L26-L49) now distinguishes the unchanged engine/graph path from the owed transport adapter. [Accepted costs](https://github.com/scottconverse/civiccast/blob/82416e0577e4f13d67b1b9b430016a195b1c1aa9/docs/adr/0021-native-windows-runtime.md#L51-L66) mark footprint/failure-surface reduction as measurement-dependent, and the standing-position links resolve from the repo root.

### Revision-introduced findings

These are folded into the SDR statuses above rather than assigned a verdict or gate action:

1. **Installer D1 versus matrix conflict** (`SDR-004`, functional): D1 changes/clears ownership where required; the matrix still demands selector untouched for every cross-uninstall.
2. **Promised rollback-failure proof row is absent** (`SDR-005`, proof gap): prose calls it a matrix row, but the matrix omits it.
3. **Migration interlock versus AC5 conflict** (`SDR-007`, functional): D1 prevents the start/write; AC5 expects the write and detects it later.
4. **Non-CI provenance lane was removed rather than completed** (`SDR-006`, proof integrity): a self-described artifact hash does not establish where/by whom it ran, and “not attestations” discards the charter's integrity/provenance option without a replacement.

One additional material issue was newly identified in Round 3 but inherited from v2 rather than introduced by the v3 diff: **the committed exact-SHA/JUnit-run metadata loop in SDR-006 has no realizable fixed point.** Round 2 missed this interaction; this record corrects that miss rather than hiding it.

The remaining “Decisions (v2)” labels in v3 documents are batched Minor wording under the owner decision. They do not affect this closure result.

### Round 3 ADR-0021 auditor position

**Auditor half: DOES NOT PASS at `82416e0577e4f13d67b1b9b430016a195b1c1aa9`.**

The ADR's own prose/path defects are closed, but it adopts two companion contracts that are still materially incomplete: dual-runtime exclusion permits native start when its WSL lifetime probe errors, and the claims rule has a self-referential exact-SHA/run binding plus incomplete mandatory inputs and non-CI provenance. Owner merge remains the other half; this review does not substitute for it.

**WS3 implementation: NOT UNBLOCKED against v3.** Close `SDR-006` before implementing the verifier so its schema and trust root do not have to be redesigned after code exists. This statement addresses design readiness only and does not inspect or alter the separate WS2 audit/gate.

### Round 3 evidence and boundary

- Resolved `origin/program/native-windows` and the requested abbreviated SHA to exact commit `82416e0577e4f13d67b1b9b430016a195b1c1aa9`; its sole parent is the Round-2 subject `397dad2c579328aa09479692606231253c172435`.
- Created a separate detached worktree, confirmed `HEAD` and a clean state, read all nine final documents, and read the complete six-file v3 diff. `git diff --check` passed.
- Rechecked the actual worker command grammar: `swap`, `reload`, `caption`, and `stop` are all commands; thus generic graph/role replay does not answer the Round-2 caption/stop criterion.
- Verified every ADR-referenced spike/spec path resolves and the three stale ADR forms from Round 2 no longer occur.
- Reconciled the charter, owner severity decision, owner testing policy, audit protocol, and the exact Round-2 acceptance-to-close bullets.
- Rechecked Microsoft primary documentation: named-pipe default DACL/data/create-instance semantics, `FILE_FLAG_FIRST_PIPE_INSTANCE` failure behavior, and machine-scope DPAPI. The v3 supervisor corrections align with those platform contracts.
- Review method: the five audit lenses were applied serially because this was a targeted design-record append, with adversarial proof-contract review layered over them. UI/visual runtime is not present in this document-only scope; operator-visible uninstall/error behavior was reviewed as UX.
- Confidence: static/source design review with state-machine counterexamples and primary-platform-contract verification. No implementation, service-session, hardware, clean-machine, or gate proof is claimed.
- PR #292 remains only the dashboard pointer venue. No WS2 source, verdict, CI result, or gate state was reviewed or changed.

## Round 4 — closure assessment at `c7022920`

**Assessment date:** 2026-07-17

**CivicCast subject:** `c7022920d4e56c100287a78f4c2a233236a0667e` on `program/native-windows`

**Compared with:** Round 3 subject `82416e0577e4f13d67b1b9b430016a195b1c1aa9`

**Review type:** exact Round-3 acceptance-criterion closure review and v4-delta sweep; not a slice audit, not a verdict, and no gate action

**Execution posture:** `sandbox=danger-full-access`, `approval_policy=never`

**Detached worktree:** `C:\Users\scott\Desktop\CODE\_audit-worktrees\civiccast-specs-r4-c7022920` (clean)

### Round 4 conclusion

V4 genuinely closes six of the seven Round-3 remainders. The dual-runtime guard, worker command semantics, uninstall matrix, upgrade freeze/rollback matrix, migration controls, and firewall proof venue now meet the smallest closures previously stated. The four SDRs already closed in Round 3 remain closed.

The revised claims rule also makes an important correction: definitions remain in the source commit while results resolve at verification time, so the impossible self-referential commit/run fixed point is gone. `SDR-006` nevertheless remains **PARTIALLY CLOSED** because the two result-resolution modes still lack source/authority bindings that an implementation can enforce correctly, and the displayed schema omits data that its completeness checks require.

| Status | Count |
|---|---:|
| CLOSED | 10 |
| PARTIALLY CLOSED | 1 |
| NOT CLOSED | 0 |

**The auditor half of ADR-0021's rung-3 review does not pass at `c7022920d4e56c100287a78f4c2a233236a0667e`.** ADR-0021 explicitly ships the claims-evidence rule with the architectural decision, and that rule still has a material proof-authority defect. **WS3 implementation is not unblocked against v4.** Its next revision should close `SDR-006` before the verifier schema, workflow contract, and external resolver are implemented.

### Closure matrix

| Finding | Round 4 status | Assessment against the Round-3 acceptance-to-close bullets |
|---|---|---|
| SDR-001 | **CLOSED** | A2 unreadable/timeout is now explicitly non-authorizing, enters `blocked_probe_unavailable`, retries, and never starts transmission. AC9 supplies the requested both-direction keeperless-live-WSL control. |
| SDR-002 | **CLOSED** | Unchanged from Round 3. The explicit named-pipe/global-object security and pre-creation degraded-path contract remains intact. |
| SDR-003 | **CLOSED** | All four actual verbs now have explicit delivery/replay semantics: `reload`/`swap` converge at least once, `caption` is at most once and never replayed, and `stop` is exactly-once-effective with observed process exit as ground truth. The named failure controls are parameterized across all four. |
| SDR-004 | **CLOSED** | The lifecycle matrix is split into the four requested ownership cases—inactive removal, active-with-survivor transfer, active-sole selector clear, and refused/cancelled transfer—with survivor operability required where applicable. It no longer contradicts D1. |
| SDR-005 | **CLOSED** | Upgrade now acquires the shared D7a interlock before drain, all native start paths must honor it, step 6 is an explicitly read-only/no-worker maintenance health mode, and release occurs only at commit. The matrix now includes both injected rollback-restore failure and concurrent manual/SCM start attempts. |
| SDR-006 | **PARTIALLY CLOSED** | The fixed-point loop, typed claim inputs, fixture declaration, control failure semantics, and self-reference/forgery mutations are materially improved. Three enforceability gaps remain in D2/D3/D7: PR source-SHA identity, canonical external authority, and the absent machine source for expected jobs/shards and collection floor. |
| SDR-007 | **CLOSED** | AC5 now asserts start-path refusal and zero writes; AC5b separately proves the final comparison catches an interlock bypass. AC5c–e enumerate the exact operator-added-config, mutation-during-hash, and reused-identity controls requested in Round 3. |
| SDR-008 | **CLOSED** | Unchanged from Round 3; no design acceptance item regressed. Implementation proof remains owed by its future slice. |
| SDR-009 | **CLOSED** | AC-S1 now binds firewall proof to a second clean class-6 Sandbox/VM peer and explicitly leaves physical-network/device proof to LPM, matching the owner testing boundary. |
| SDR-010 | **CLOSED** | Unchanged. Decision-state headers and the owner-acceptance register continue to leave owner decisions with Scott. |
| SDR-011 | **CLOSED** | Unchanged. ADR premise scoping, measurement-dependent claims, and repo-relative path corrections remain intact. |

### Remaining SDR-006 design findings

#### 1. Same-run mode binds the wrong SHA on pull-request workflows

[Claims D3 says `GITHUB_SHA` equals the commit being verified](https://github.com/scottconverse/civiccast/blob/c7022920d4e56c100287a78f4c2a233236a0667e/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L39-L51). That is false for GitHub Actions `pull_request` events: GitHub defines `GITHUB_SHA` as the PR merge-branch commit, while the source head is `github.event.pull_request.head.sha`. The current repository demonstrates the distinction: PR #292 resolves to head `90c408d08ae534d939f1755443c85ff77dcc3529` and merge ref `abcc1abde7b8206ec7685480b7466e149b58e9ab`.

Consequence: a verifier built literally from v4 can bind a passing result to a synthetic merge commit while labeling it as evidence for the source head, or reject a correctly checked-out head because runtime `GITHUB_SHA` differs. This is a functional proof-identity defect, not wording.

Smallest closure: define `source_sha` by event and bind it to the exact checkout. For `pull_request`, use and explicitly check `github.event.pull_request.head.sha`; if merge-result testing is also required, record `merge_sha` separately and never substitute it for the audited source SHA. Add a PR fixture/control where head and merge SHA differ and prove the wrong identity is rejected.

#### 2. External evidence is located, but not canonically authorized

[External-evidence mode](https://github.com/scottconverse/civiccast/blob/c7022920d4e56c100287a78f4c2a233236a0667e/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L52-L62) calls audit-control “append-only, owner-controlled” and resolves a record “by SHA via the audit-control remote.” The authoritative protocol says the opposite trust nuance: audit-control is a **process boundary, not a credential boundary**; the shared PAT does not authenticate actors and history is inspectable but not protected. Verdict authority works because a canonical path, exact source SHA, exact audit-control commit/blob, structured fields, thread identity, repository origin, and fail-closed lookup are validated—not because an arbitrary file is present in the repository.

V4 specifies none of those external-record selection/authority rules. It does not define a canonical path/schema, uniqueness, accepted reviewer/owner authority, exact governance commit/blob, repository-origin validation, or fork/wrong-branch/ambiguous-record rejection. A coder-authored record pushed with shared credentials could therefore satisfy the written rule without independent review.

Smallest closure: define one canonical external-evidence path and structured schema keyed by source SHA and claim/control ID; bind it to a named 40-hex audit-control commit and blob; validate the exact `scottconverse/civiccast-audit-control` origin and commit object; require the structured independent reviewing authority/attestation appropriate to the confidence class; reject zero, duplicate, forked, wrong-SHA, wrong-blob, or unauthoritative records with exit 2. Reuse the completion hook's canonical-validation rigor rather than only its repository name.

#### 3. Completeness checks have no declared machine source

D3 requires the run's job set to match expected jobs/matrix shards and the JUnit count to meet a “recorded floor,” but [D2's committed definition shape](https://github.com/scottconverse/civiccast/blob/c7022920d4e56c100287a78f4c2a233236a0667e/.agent-runs/native-windows/specs/spec-claims-evidence-rule.md#L23-L38) declares neither the expected job/shard set nor a collection floor. D7 only says the verifier depends on “the test job,” which does not make all other required jobs complete before a whole-workflow comparison. D2 also embeds the schema inside the `verifier` role while D5 describes `verifier` and `schema` as separate typed roles, leaving the promised missing-role mutation structurally ambiguous.

Smallest closure: put `collection_floor` and the required job/matrix set in the committed typed definition, or define one deterministic derivation from the bound workflow and test manifest. Make the verifier depend on every producer it claims to assess (or narrow the claimed set to its completed dependencies). Reconcile `schema` as either its own required role or a named member of `verifier`, then mutate the actual schema shape in the acceptance test.

### V4 new-finding sweep by document

- `spec-dual-runtime-guard.md`: no new material finding; its A2 asymmetry is now conservative and testable.
- `spec-supervisor.md`: no new material finding; per-verb replay and clean-peer venue are internally consistent. Implementation still owes the platform/runtime proofs already named.
- `spec-installer-lifecycle.md`: no new material finding; D3 and the proof matrix now tell one upgrade/rollback story, and the four uninstall rows are consistent.
- `spec-migration-contract.md`: no new material finding; the interlock is primary and final comparison is explicitly defense in depth.
- `spec-claims-evidence-rule.md`: the three material gaps above are introduced or exposed by the v4 trust-model rewrite and remain within `SDR-006`.
- Unchanged `spec-packaging-closure.md`, `POST-FABLE-PLAYBOOK.md`, specs `README.md`, and ADR-0021 were checked for regression against the Round-3 closure record; no new material issue was found. The owner-acceptance register remains controlling.

No standalone wording finding is raised. Residual version-label/prose nits remain batched and non-blocking under the owner's severity calibration.

### Round 4 ADR-0021 auditor position

**Auditor half: FAIL at `c7022920d4e56c100287a78f4c2a233236a0667e`.** The native-runtime architecture and all non-claims SDRs now meet this review's design bar, but ADR-0021 expressly ships the claims-evidence rule. Passing the ADR while its proof-authority mechanism can bind the wrong PR SHA or accept an under-specified external record would contradict the ADR's own honesty boundary. Scott's owner-merge half remains separate and has not occurred through this review.

**WS3 implementation: NOT UNBLOCKED against v4.** Close the three `SDR-006` bullets above first. They change the verifier's core data model, workflow dependencies, and trust resolver; deferring them would predictably force a redesign after implementation.

### Round 4 evidence and boundary

- Resolved `origin/program/native-windows` and the requested abbreviated SHA to exact commit `c7022920d4e56c100287a78f4c2a233236a0667e`; its sole parent is Round-3 subject `82416e0577e4f13d67b1b9b430016a195b1c1aa9`.
- Created a fresh detached worktree, verified exact `HEAD`, origin, and clean state, read the five-file v4 diff and final changed documents, and reconciled every Round-3 acceptance-to-close bullet. `git diff --check` passed.
- Rechecked the unchanged companion documents and ADR against the Round-3 record; no closed SDR regressed.
- Falsified the unconditional `GITHUB_SHA` premise against both [GitHub's official pull-request event contract](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request) and this repository's live PR #292 head/merge refs.
- Reconciled external-evidence authority against audit-control `AUDIT_PROTOCOL.md` sections 2 and 8 and the charter's independent-review/attestation boundary.
- Applied the five review lenses serially with an adversarial proof-contract pass. There is no UI/visual runtime in this document-only delta; operator-visible degraded, uninstall, maintenance, and recovery states were reviewed as UX contracts.
- Confidence: static/source design review with exact-SHA binding, state-machine reconciliation, a live Git ref counterexample, and primary GitHub platform documentation. No implementation, CI execution, service-session, clean-machine, hardware, LPM, slice verdict, or gate proof is claimed.
- PR #292 remains only the dashboard pointer venue. No WS2 source, verdict, CI result, merge state, or gate state was reviewed or changed.
