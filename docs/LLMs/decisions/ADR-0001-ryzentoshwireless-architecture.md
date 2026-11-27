# ADR-0001: RyzentoshWireless.kext architecture and what it is built from

Status: **proposed** (2026-11-27). Decided by: Project Lead (claude-opus-5-5/high), with review by claude-fable-5-2. The
owner decisions it rests on are already recorded in the control plane's `docs/LLMs/DECISIONS.md` (one repo, one kext;
GPL-3.0-only; MtkWirelessWifi study-only; the bluetoothd patch as the named essential workaround).
Evidence: [RB-W1](../re/RB-W1-wireless-architecture.md). Issue: [WIRELESS#3](https://github.com/ownback/Ryzentosh-WIRELESS/issues/3).

## Context

- Hardware: RZ616 / MT7922 Wi-Fi (PCI `14c3:0616`, subsystem `105b:e0cd`) and its Bluetooth half (USB `0489:e0e2`), one
  combo chip. Today's EFI loads MtkWireless 1.1.3 (Wi-Fi + BT) + BlueToolFixup + Lilu.
- Apple's model on 25G241 (RB-W1 §2): one `IO80211Controller` with one interface per role (infra, AWDL, NAN, SoftAP, low
  latency), with an IOSkywalk data path. The AWDL schedule runs in the chip's firmware.
- MediaTek firmware has no AWDL engine. Every MediaTek driver runs AWDL on the host with remain-on-channel grants (RB-W1 §3).
- Measured: MtkWireless 1.1.3 is the best candidate on this machine (97.9/50.7 Mbps, 308 RPM, Tx 867, 0 % ping loss in
  30 min). AirportMtkWlan 2.3.3 was worse and was rolled back (EFI PR #10). No candidate transfers AirDrop with the AP on
  DFS channel 52 (RB-W1 §4.1).
- License: the repo is GPL-3.0-only. MtkWirelessWifi (incl. `rust/mtk-wlan`) is GPL-2.0-only and study-only.
  AirportMtkWlan's GPL-3.0 grant covers code inherited from GPLv2 projects (RB-W1 §5). The hardware-layer primaries are
  ISC: Linux mt76 `mt7921`/`mt792x` and `btmtk.c`.

## Decision drivers

1. Parity: AirDrop, Handoff, Clipboard, Sidecar, Continuity Camera and iPhone Bluetooth must work the way a Mac does it,
   including with the AP on a DFS channel, with no boot-args.
2. Every line traceable to a source we may legally use in GPL-3.0-only code.
3. Measured on the owner's machine against the MtkWireless 1.1.3 baseline; nothing ships that measures worse.
4. Apple's driver-quality bar (`apple-driver-quality.md`): IOKit lifecycle, work loops, no sleeps as synchronization,
   os_log, on-demand diagnostics.

## Options considered

| | Option | Verdict |
|---|---|---|
| A | Fork AirportMtkWlan as the code base | **Rejected.** It measured worse on this machine, and its GPL-3.0 grant covers GPLv2 code (SimpleMtkWlan, itlwm) whose "or later" status isn't shown per file. We'd inherit both problems. |
| B | Fork MtkWireless | **Not allowed** for Wi-Fi (GPL-2.0-only, owner decision). Its BT kext is GPL-3.0-only, but its C++ files say they're based on zxystd/laobamac code with unknown licenses, and its logic lives in Rust. |
| B' | Ask MtkWireless's author to relicense `mtk-wlan` as GPL-2.0-or-later | Not enough on its own: the vendored SimpleMtkWlan HAL isn't the author's to relicense. It's an outward-facing request, so it needs the owner's approval. **Kept as an optional later step**, not on the critical path. |
| **C** | **Our own driver.** Hardware layer ported from ISC/BSD primaries (mt76, btmtk, OpenBSD net80211/mwx); our own IO80211 front end from KDK RE; our own host AWDL engine; MtkWireless 1.1.3 as the behaviour reference and the A/B baseline | **Chosen.** |

"Base" in the owner's plan (ROADMAP M5: "the one that gets furthest and works best") is therefore **MtkWireless 1.1.3 as
the reference and baseline**. It is the bar to beat, and a source of facts (what the hardware and macOS needed, cited by
commit and path), never of code.

## Decision

### 1. One bundle, one binary

`RyzentoshWireless.kext` (bundle id `com.ryzentosh.wireless`) has one executable and these IOKit personalities:

| Personality | Matches | IOClass |
|---|---|---|
| Wi-Fi | `IOPCIPrimaryMatch` `0x061614c3` (the tested subsystem is `105b:e0cd`; other MT7922 IDs only with tests) | `RyzentoshWLANController` (`IO80211Controller` subclass) |
| Bluetooth | `IOUSBHostDevice`, `idVendor 0x0489`, `idProduct 0xe0e2` | `RyzentoshBTLoader` |
| bluetoothd patch | `IOResources`, Lilu plugin entry | `RyzentoshBTPatcher` |

`OSBundleLibraries`: the IO80211Family, IOSkywalkFamily, IONetworkingFamily, IOPCIFamily and IOUSBHostFamily libraries
Apple's driver uses (RB-W1 §2.1), plus Lilu (`as.vit9696.Lilu`). The personalities start independently. If the IO80211
ABI guard (component 3) declines on an unknown build, only Wi-Fi stays down and Bluetooth still runs. `MinKernel` /
`MaxKernel` = `25.0.0` / `25.99.99`; macOS is pinned to 25G241.

Trade-off, accepted on the owner's "one binary" decision: a crash in one part takes down the whole kext. That is the same
as Apple's single-binary core driver.

### 2. Components (source layout `src/<component>/`)

| # | Component | Responsibility | Built from |
|---|---|---|---|
| 1 | `chip` | MT7922 PCIe bring-up, WFDMA rings, MCU commands, firmware download, driver/firmware ownership, chip reset (SER) and the **reset coordinator** shared by Wi-Fi and BT (a chip reset takes BT down too) | port of mt76 `mt7921/pci.c`, `mt7921/mcu.c`, `mt792x_core.c`, `mt792x_dma.c`, `mt76_connac_mcu.c` (ISC) |
| 2 | `wlan/mac` | TX/RX descriptors, BA reorder, rate control feedback, keys, power save, hw_scan, **channel contexts and ROC** (CNM) | mt76 `mt7921/mac.c`, `mt792x_mac.c`, `mac80211`-facing logic re-expressed (ISC) |
| 3 | `io80211` | `RyzentoshWLANController`, infra interface (`IO80211InfraProtocol`), AWDL interface (`IO80211AWDLProtocol`), Skywalk queues. Our own headers generated from the 25G241 KDK, with a **class-size ABI guard** at probe (the WTLWireless pattern, ISC, re-implemented) | KDK RE (REA), Apple's class choice from AppleBCMWLANCoreMac |
| 4 | `wlan/mlme` | 802.11 state below the Family's WCL: auth/assoc frames, RSN (WPA2-PSK, WPA3-SAE, PMF), the supplicant | OpenBSD net80211 (ISC/BSD), IEEE 802.11-2020 for SAE |
| 5 | `regd` | real country and regulatory limits: CLC/SKU from the chip, 802.11d, mapping to `apple80211` locale. Needed for correct AWDL channel choice (RB-W1 §3.4) | mt76 `mt7921_mcu_set_clc` (ISC) |
| 6 | `awdl` | **host AWDL engine**: mirrors the Family's schedule selectors (sync params, channel sequence, election, presence), executes availability windows through component 2's ROC/channel contexts, AP power-save handshake around off-channel windows, DFS and regulatory channel policy, peer data path and the peer signals the Family needs (no driver-installed routes, RB-W1 §3.4) | Apple behaviour from KDK RE; AWDL protocol facts from public research (seemoo-lab OWL papers; `owl` is study-only); MtkWireless docs as measured facts |
| 7 | `bt` | USB firmware download (WMT) and the controller's power and reset; then IOBluetoothFamily's own USB transport drives it | port of Linux `btmtk.c` (ISC) |
| 8 | `bt/hci` | research first (issue W8): make the controller answer the Apple vendor commands Continuity depends on (advertising, scan filters, IRK lists), e.g. by mapping to MediaTek's Microsoft-extension monitor. Only built if RB evidence shows it's needed | to be decided by W8 |
| 9 | `bluetoothd` | **the named essential workaround**: Lilu user-space patches that make bluetoothd accept a non-Apple controller. Minimal patch set, each site matched by signature against the pinned build, fail-closed (unknown build → no patch, Bluetooth stays usable without Continuity), behind the feature flag `rzw-btd-patch` (default on), tracked by its own issue | our own RE of the 25G241 bluetoothd (REA). BlueToolFixup (BrcmPatchRAM: GitHub reports GPL-2.0) and MtkWirelessBT are study-only until the license-auditor traces them |
| 10 | `common` | os_log subsystem `com.ryzentosh.wireless` (categories per component), ioreg diagnostics published on demand, `IOReturn` everywhere | ours |

Language: **C++ in Apple's IOKit subset only.** No Rust in the kernel: one toolchain, Apple's style, and the reviewer can
read everything. Pure logic (firmware header parsing, AWDL schedule maths, channel and regulatory policy, patch signature
matching, BA reorder) lives in files without kernel dependencies, so host-side unit tests in `tests/host/` can build it.

Firmware: the MT7922 Wi-Fi and BT images from linux-firmware go into the bundle's `Resources/` with `LICENCE.mediatek`
(redistribution allowed by that licence; license-auditor confirms the exact text). A Mac also ships its Wi-Fi firmware
inside the OS (`BCMWLANFirmware*_Hashstore.kext`).

### 3. No boot-args, no runtime crutches

- Shipped code reads no tuning boot-args. Debug knobs exist only in DEBUG builds, as ioreg properties.
- No driver-installed routes or neighbour entries. Missing peer signals are fixed at their source (component 6).
- The only exception is component 9, plus the two NVRAM keys bluetoothd needs for a third-party controller
  (`bluetoothInternalControllerInfo`, `bluetoothExternalDongleFailed`, set in Ryzentosh-EFI). Both are part of the same
  named workaround. Its tracking issue documents why a Mac doesn't need it.

### 4. License plan

| Material | Use | Obligations |
|---|---|---|
| Linux mt76 (`mt7921/*`, `mt792x_*`, `mt76_connac*`), Linux `btmtk.c` (ISC, v6.18) | port | keep copyright + ISC text in `NOTICE.md`; `PROVENANCE.md` row per file (commit, path, function) |
| OpenBSD net80211 / mwx(4) (ISC/BSD) | port | the same |
| WTLWireless (ISC), mt7925-macos (BSD-3-Clause-Clear) | port patterns, mostly ideas; copy only with notices | the same |
| AirportMtkWlan | copy only files with their own GPL-3.0-compatible SPDX line, after the license-auditor traces them; otherwise study | per file |
| MtkWireless Wi-Fi (incl. its GPL-2.0-or-later AWDL files, which depend on GPL-2.0-only code), MtkWirelessBT C++ (zxystd/laobamac lineage), BrcmPatchRAM/BlueToolFixup, SimpleMtkWlan, AirportMtk-AWDL/-macOS, owl | **study only**: facts with citations, then our own code | none in our tree |
| `Airport/*` RE headers from the itlwm line | **not used**; we generate our own from the KDK | — |
| Apple binaries | RE facts only (names, offsets, behaviour, build) | never redistribute |
| MediaTek firmware | bundled with `LICENCE.mediatek` | notice |

Every source file starts with `SPDX-License-Identifier: GPL-3.0-only`. Ported files keep the original copyright lines
under it. The license-auditor signs off on each PR that adds third-party material.

### 5. Order of work

1. ABI headers and the guard (W2), then chip bring-up and firmware (W3) with host tests.
2. Infra STA to baseline parity (W4, W5): it must reach the MtkWireless 1.1.3 numbers before anything else ships.
3. Bluetooth loader + bluetoothd patch (W7) can run in parallel with step 2. It replaces MtkWirelessBT + BlueToolFixup.
4. AWDL engine (W6) after the infra STA, then the Continuity features one by one.
5. Each step is one candidate EFI through the Orchestrator, compared with the step-0/1.1.3 baselines. Anything worse → no
   merge.

## Test plan per compatibility-matrix row

The baseline is MtkWireless 1.1.3 on this machine unless the row says otherwise. Host tests run on every build; device
tests run on each candidate boot.

| Matrix row | Parity target (a real Mac) | Device test | Pass |
|---|---|---|---|
| Wi-Fi | Native Wi-Fi menu, join WPA2/WPA3, 802.11ax 80 MHz | `networkQuality -v` ×3, `wdutil info`, 30-min gateway ping, 1-h idle | ≥ 97.9/50.7 Mbps, ≥ 308 RPM, Tx ≥ 867, 0 % loss, no link drop |
| Wi-Fi on DFS AP | Joins and stays on ch 52 | the same on ch 52 | the same |
| AirDrop | Both ways while associated, also on a DFS AP | 10 photos + one 1 GB file each way, AP on ch 52 and on ch 36–48 | 20/20 small, both large; no AP disassociation (`log` + gateway ping) |
| Handoff | Safari tab both ways | 10 tries each way | ≥ 19/20 |
| Universal Clipboard | Text and image both ways | 10 copies each way, link type logged | ≥ 19/20 |
| AirPlay to Mac | Music, video, mirroring | 5 min each | no drop |
| Continuity Camera (wireless) | iPhone as webcam without a cable | FaceTime preview 5 min | image, no drop (needs GPU decode, see RB-W1 §4.4) |
| Sidecar / Universal Control / iPhone Mirroring | where the Mac model supports them (MacPro7,1: check per feature) | 10 min each | works like on a Mac of the same model |
| Instant Hotspot | Mac joins the iPhone's hotspot from the Wi-Fi menu | 5 joins | 5/5 |
| Bluetooth: accessories | AirPods A2DP/AAC, HFP mic, keyboard/mouse | 30 min audio, 5-min call | no drop, wideband mic |
| Bluetooth: iPhone | Pair and connect from either side | pair, unpair, re-pair ×3 | 3/3, bluetoothd never relaunches |
| BLE advertising | bluetoothd advertising never fails | `log` count of `Could not start advertising` / `Failed to add rule` per hour | 0 |
| Sleep/wake | Wi-Fi and BT survive ×3 | waits for M1.5 (sleep is off on this system) | — |
| Boot | Clean load, no panic | boot report | no new flags |

Host tests (`tests/host/`): firmware header parsing (Wi-Fi patch/RAM, BT), MCU TLV encoders against mt76 layouts, BA
reorder, regulatory mapping, AWDL availability-window maths and channel policy (DFS, country, AP channel), ABI-guard
class-size table, bluetoothd signature matcher against the pinned binary's bytes (read locally, never committed).

## First implementation issues

| # | Title | Depends on |
|---|---|---|
| W2 | IO80211/IOSkywalk 25G241 ABI: generate headers from the KDK, class-size guard, host test | — (REA) |
| W3 | MT7922 chip bring-up: PCIe, WFDMA, MCU, Wi-Fi firmware download, reset coordinator (mt76 port) | W2 for the kext shell |
| W4 | Infra STA: scan, join WPA2/WPA3-SAE/PMF, Skywalk data path; meet the 1.1.3 baseline | W3 |
| W5 | Regulatory: real country/CLC, `apple80211` locale | W3 |
| W6 | AWDL engine: host schedule over ROC, AP power-save handshake, DFS-AP case, peer data path without routes (RB-W1 Q2/Q3) | W4, W5 |
| W7 | Bluetooth: btmtk firmware loader for `0489:e0e2` + bluetoothd patch (named workaround, flag `rzw-btd-patch`, fail-closed) from our own 25G241 RE | — |
| W8 | RB: Apple Bluetooth vendor commands that Continuity needs, and whether the MediaTek controller can serve them (RB-W1 Q4) | W7 |
| W9 | iPhone Bluetooth connection: root cause from an HCI trace (owner test T2) and the `Unknown chipset 0` relaunch (RB-W1 Q5) | owner test T2 |
| W10 | On-device test scripts for the table above (networkQuality, ping, AirDrop/Clipboard trial logger, BT log counters) | — |
| W11 | Licence audit: AirportMtkWlan per-file provenance, MtkWirelessBT/BlueToolFixup lineage, `LICENCE.mediatek` text | — |

Owner tests that need no reboot and can run now (RB-W1 §4): **T1** AirDrop with the AP on a non-DFS channel, **T2** iPhone
pairing with an HCI trace, **T3** Clipboard tries with link-type logging.

## Consequences

- Longer path to a first release than forking, but every file is legally clean and written to our bar.
- The live system keeps MtkWireless 1.1.3 until each RyzentoshWireless part beats it on hardware (EFI-END-STATE: then
  MtkWirelessWifi, MtkWirelessBT and BlueToolFixup go away together).
- AWDL on a single radio without a firmware engine may not reach a Mac's AirDrop throughput. The parity bar is "works in
  every case a Mac works". Throughput is measured and reported, not asserted.
- WIRELESS#1 (`amtk_awdl_offchan=0`) closes when W6 passes the DFS-AP test and the arg leaves the EFI.
