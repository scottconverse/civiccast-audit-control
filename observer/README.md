# Release-Truth Observer

A scheduled, **read-only** watcher that runs CivicCast's release-truth checker
(`scripts/policy/check_release_truth.py` at the pinned program-branch commit)
against live GitHub release state and the public `main` README, and files or
updates a drift issue **in this repository** when claims and reality disagree.

Authority: **none.** This observer gates nothing, blocks nothing, and has no
role in any release — the WSL rc line ships entirely on Scott's and the
rc-line coder's authority. Drift issues here are alarms for humans, in the
audit-control repo only, producing zero noise in the civiccast repo.

Pinning: the workflow fetches the checker and manifest from an exact CivicCast
commit (`CHECKER_PIN` in the workflow). Update the pin when a newer
release-truth slice merges to `program/native-windows` (and eventually `main`).
A stale pin fails visible (drift issue notes the pin), never silent.

The README it checks is `main`'s — the surface visitors actually see — while
the manifest comes from the program branch until the rc line adopts it.
An expected consequence: when the rc line publishes a new release (e.g. rc15),
the observer files an "unknown release" drift issue until the manifest gains
an entry. That alarm is the mechanism working, not a malfunction.
