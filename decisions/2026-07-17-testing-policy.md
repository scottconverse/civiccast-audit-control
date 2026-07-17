# Owner decision — testing policy for the native line (2026-07-17)

Decided by Scott Converse (owner):

1. **No 24-hour soaks in this program.** A limited number of 4-hour soaks
   are authorized to shake out obvious bugs. The software has largely
   proven itself on the WSL line; duration-proof beyond 4h is not a gate.
2. **All real-world testing happens at LPM** (the real PEG station).
   This program does software only. The repo's own contract-lab proof
   levels are the rule: everything up to `software-lab-proven` /
   `api-contract-proven` is provable here (including via the virtual lab
   in civiccast/control_room and the agenda-system fixtures);
   `station-device-proven` belongs exclusively to LPM.
3. Audit confidence classes map accordingly: class 5 (hardware/live-peer)
   and class 7 (soak beyond 4h) are NOT gate requirements for the native
   beta handed to LPM; class 6 (pristine machine) remains required and is
   satisfiable with clean VMs/Windows Sandbox (software-only).

The auditor applies these bounds when judging gate advancement; demanding
proofs beyond them is out of calibration.
