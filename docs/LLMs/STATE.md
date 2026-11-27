# STATE: Ryzentosh-WIRELESS

Updated: 2026-11-27 ~22:50 by the Project Lead for WIRELESS#3 (claude-opus-5-5/high).

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

## In flight
- Draft PR for RB-W1 + ADR-0001 (branch `research/3-wireless-architecture`); Fable review pending.

## Next step
Fable review → fix → mark ready → Orchestrator decides the ADR. Then file issues W2-W11. Owner tests T1-T3 (no reboot) whenever the owner has time.

## Blockers
None.
