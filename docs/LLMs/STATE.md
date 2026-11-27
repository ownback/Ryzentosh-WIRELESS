# STATE: Ryzentosh-WIRELESS

Updated: 2026-11-27 ~23:20 by the Project Lead for WIRELESS#3 (claude-opus-5-5/high).

## Phase
Research → ADR (RB-W1, WIRELESS#3). No kext code yet.

## Done
- RB-W1 findings: `docs/LLMs/re/RB-W1-wireless-architecture.md` (Apple's AppleBCMWLANCoreMac model from the 25G241 KDK, candidate comparison, root causes of the gaps, license facts).
- ADR-0001 proposed: `docs/LLMs/decisions/ADR-0001-ryzentoshwireless-architecture.md` (option C: own driver from ISC/BSD primaries; MtkWireless 1.1.3 = reference baseline, study-only; one binary; components; license plan; test plan; issues W2-W11).
- MtkWireless cloned study-only into the control plane's `repositories/MtkWireless` (commit 424f976, marker `.RYZENTOSH-STUDY-ONLY`).

## Key findings
- AirDrop "discovery, no transfer": AP on DFS ch 52 + MtkWireless never uses DFS for AWDL + `amtk_awdl_offchan=0` → AWDL has no channel. Owner test T1 (AP on ch 36-48, no reboot) confirms.
- bluetoothd on this boot: `Unknown chipset 0` fault + relaunch at 21:42 during a Bluetooth Settings session; 245 failed Apple scan-filter VSCs and 11 advertising failures in 50 min. iPhone BT not root-caused (owner test T2: HCI trace).
- Continuity Camera wired: not WIRELESS (VideoToolbox reports HEVC HW decode true, H.264 false) → GPU.
- AirportMtkWlan's GPL-3.0 grant covers GPLv2-inherited code; copy only per-file-traced files. mt76 mt7921/mt792x and btmtk.c are ISC (v6.18). BrcmPatchRAM (BlueToolFixup) is GPL-2.0 per GitHub → study-only until audited.

- Bluetooth: bluetoothd drives the controller from user space (USB user clients), so a kext can't sit in the HCI path; Apple-VSC emulation would need more bluetoothd patches or a proxy device (outside the owner's exception → W8 ends in an owner decision).
- owl (seemoo-lab): the Orchestrator lifted it on 2026-11-27. Its 34 GPL-3.0-or-later project files can be copied per file (marker `.RYZENTOSH-COPY-PER-FILE`); the vendored files keep their own terms.

## In flight
- PR https://github.com/ownback/Ryzentosh-WIRELESS/pull/4 (ready for review). Fable: review 1 CHANGES-REQUESTED → rev 2 → re-check APPROVE. ADR-0001 accepted; waiting for the Orchestrator's squash-merge.

## Next step
After the ADR is accepted: file issues W2-W11 from the ADR table, then start W2 (ABI headers, REA) and W7 (BT loader). Owner tests T1-T3 (no reboot) whenever the owner has time; T1 (AP on ch 36-48, AirDrop) confirms the AirDrop root cause.

## Blockers
None. ADR-0001 was accepted on 2026-11-27 (option C confirmed by the owner via the Orchestrator).
