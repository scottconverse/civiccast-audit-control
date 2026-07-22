# Owner decision — egress work-dir at-rest protection: locked folder, no encryption (2026-07-21)

Decided by Scott Converse (owner).

## Context

Sink-connection passphrases are resolved into the serialized playout-graph JSON that
the egress worker reads (they are written as plaintext values in the file, not a
reference). At-rest protection today is best-effort only: the `chmod(0o600)` in
`_write_graph_file` is a documented no-op on Windows (it toggles only the read-only
attribute, sets no NTFS ACL), and the default work-dir under a `LocalSystem` service
is `%LOCALAPPDATA%` = the SYSTEM profile, which is not one of the paths D4's ACL table
names. So on a native Windows install these files land outside the promised ACL.

## Decision

1. **Lock the folder; do not encrypt.** Owner's threat assessment: these machines sit
   in locked datacenter rooms, not shared workspaces, and are not shared with
   unauthorized people — so a directory ACL restricting reads to SYSTEM+Administrators
   is sufficient at-rest protection. Encryption-at-rest of the passphrases is **not**
   required at this time.
2. **Build-time acceptance criterion (WS5, hard).** The supervisor MUST set
   `CIVICCAST_EGRESS_WORK_DIR` into the `ProgramData\CivicCast\data` tree so the egress
   graph/reload files land where D4's ACL applies and where AC-S4's icacls evidence
   actually covers them. `core.py`/`service.py` own this; it is a required AC for the
   WS5 slice, not optional.

## Status

The plaintext-at-rest property in the shipping product is accepted as-is under the
locked-room threat model above; the WS5 build closes the "wrong folder / no effective
ACL" gap on the native path via the AC in (2). No further remediation is scheduled.
The detailed finding is held privately (routed per `CONTRIBUTING.md` → `SECURITY.md`),
not published to the public repo.
