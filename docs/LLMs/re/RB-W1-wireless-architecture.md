# RB-W1: How 25G241 expects a Wi-Fi + AWDL + Bluetooth driver, how the candidates do it, and why the gaps exist

Issue: [WIRELESS#3](https://github.com/ownback/Ryzentosh-WIRELESS/issues/3). ADR: [ADR-0001](../decisions/ADR-0001-ryzentoshwireless-architecture.md).
Author: Project Lead, claude-opus-5-5/high, 2026-11-27. Status: research findings, ready for review.

## 0. Answers in short

1. **Apple's own model** (AppleBCMWLANCoreMac, the T2 Macs' Wi-Fi driver in the 25G241 KDK) is one controller
   (`IO80211Controller` subclass) with one interface object per role: infra (`IO80211InfraProtocol`), AWDL
   (`IO80211AWDLProtocol`), NAN (`IO80211NANProtocol`, `IO80211NANDataProtocol`), SoftAP (`IO80211SapProtocol`), plus a
   low-latency interface. Packets move through IOSkywalk submission/completion queues. The Family (IO80211Family) keeps
   AWDL peer management, election state and the DFS-proxy policy on the host, and **pushes the AWDL schedule into the
   chip's firmware** through about 50 `apple80211` AWDL get/set selectors. The Broadcom firmware runs the AWDL timing.
2. **Every MediaTek candidate uses the same class layout** (controller + `IO80211InfraProtocol` + `IO80211AWDLProtocol`,
   the AirportItlwm pattern). MediaTek firmware has no AWDL engine, so the candidates **run the AWDL schedule on the host**
   with firmware remain-on-channel (ROC) grants and an AP power-save handshake. That is the main technical risk of the
   project. It is also why AWDL quality differs so much between drivers.
3. **AirDrop "discovery yes, transfer never": root cause found (high confidence, needs one confirming test).** The AP is on
   DFS channel 52. MtkWireless never uses DFS channels for AWDL, and the boot-arg `amtk_awdl_offchan=0` keeps AWDL on the
   AP's channel. So AWDL has no channel it may use while associated. Discovery runs over BLE and works; the AWDL data path
   never opens.
4. **Handoff / Universal Clipboard flakiness:** not proven. The evidence points at Bluetooth LE advertising and scan-filter
   failures that are the same on every Wi-Fi driver tested (the Bluetooth kext was the same in all three tests), plus the
   missing AWDL data path from point 3.
5. **iPhone can't connect over Bluetooth:** not root-caused. On this boot bluetoothd logged a fault, `Unknown chipset 0`,
   and was relaunched during a Bluetooth Settings session. An HCI trace of one pairing attempt is needed (owner test T2).
6. **Continuity Camera, wired, no image:** not a WIRELESS defect (the wired path is USB). It goes to Ryzentosh-GPU (media
   decode, RB-0003g). Wireless Continuity Camera needs AWDL and waits on point 3.
7. **License:** AirportMtkWlan's top-level GPL-3.0 grant sits on code inherited from GPLv2 projects (SimpleMtkWlan,
   itlwm). Its files can't be copied without per-file provenance. The clean primary sources for the hardware layer are
   permissive: Linux mt76 (`mt7921/*`, `mt792x_*`: ISC) and `btmtk.c` (ISC). See ADR-0001 for the plan.

## 1. Sources and method

| Source | Version | How used |
|---|---|---|
| KDK 25G241 `IO80211Family.kext` | 1200.13.1, UUID 4184F404-9547-3499-A311-C1920829EA25 (loaded) | `nm -C` class and symbol lists, `strings` |
| KDK 25G241 `AppleBCMWLANCoreMac.kext`, `AppleBCMWLANBusInterfacePCIe.kext`, `IOSkywalkFamily.kext` | 25G241 KDK | the same; `Info.plist` |
| `/usr/sbin/bluetoothd` | 25G241, sha256 prefix `ceebf25e5f995444` | `strings` only |
| MtkWireless (Dino-VN) | commit `424f976` (2026-11-25, VERSION 1.1.3), study-only clone in the control plane's `repositories/MtkWireless` (marker added 2026-11-27) | source reading, docs |
| AirportMtkWlan (Arxusion) | commit `9bc7ad0` (v2.3.2 tree; the rejected release was 2.3.3) | source and CREDITS reading |
| WTLWireless | `734ca92` (0.15.5) | NOTICE, docs, file headers |
| mt7925-macos, YYavicWiFi, SimpleMtkWlan, AirportMtk-AWDL, AirportMtk-macOS | clones in `repositories/` | SPDX survey only |
| Linux v6.18 | GitHub API, first line of each file | license headers of mt76 and btmtk |
| Live system, this boot (MtkWireless 1.1.3, booted ~21:34) | unified log, ioreg, `system_profiler`, VideoToolbox | evidence for section 4 |
| Owner tests | EFI PR #10 step 0, EFI PR #13; control plane `logs-local/ewifi/`, `logs-local/ewifi2/` | measured baselines |

No REA session was opened. Symbol tables answered the structural questions. The deeper RE questions are listed in
section 6 and assigned to implementation issues, so they are answered when they are needed.

## 2. What IO80211Family and IOSkywalk expect on 25G241 (the Mac reference)

### 2.1 Bundle and matching

- Apple splits bus and core into two kexts. `AppleBCMWLANBusInterfacePCIe` drives the PCIe function. `AppleBCMWLANCoreMac`
  has `IOClass = AppleBCMWLANCore` and `IOProviderClass = AppleBCMWLANBusInterfacePCIe` (its Info.plist).
- Core libraries (`OSBundleLibraries`): IO80211Family 1.5.0, IOSkywalkFamily 1.0, IONetworkingFamily, IOPCIFamily,
  corecapture, IOTimeSyncFamily, mDNSOffloadUserClient, CoreAnalyticsFamily, IOImageLoader, AppleOLYHALMac, AMFI, corecrypto,
  plus the bsd/iokit/libkern/mach/private/unsupported KPIs.
- Firmware ships as separate kexts: `BCMWLANFirmware4364/4378/4387/4388_Hashstore.kext`.

### 2.2 Class roles

From `nm -C` on 25G241. "U" means imported from IO80211Family or IOSkywalkFamily.

| Role | Apple class (AppleBCMWLANCoreMac) | Base it imports | BSD interface |
|---|---|---|---|
| Controller | `AppleBCMWLANCore` | `IO80211Controller` (U) | — |
| Infrastructure STA | `AppleBCMWLANSkywalkInterface`, `AppleBCMWLANInfraProtocol` | `IO80211InfraProtocol` (U) → `IO80211InfraInterface` → `IO80211SkywalkInterface` | en* |
| AWDL | `AppleBCMWLANProximityInterface` | `IO80211AWDLProtocol` (U) → `IO80211VirtualInterface` | awdl0 |
| NAN (Wi-Fi Aware) | `AppleBCMWLANNANInterface`, `AppleBCMWLANNANDataInterface` | `IO80211NANProtocol`, `IO80211NANDataProtocol` (U) | nan* |
| SoftAP | `AppleBCMWLANIO80211APSTAInterface` | `IO80211SapProtocol` (U) | ap* |
| Low latency | `AppleBCMWLANLowLatencyInterface` | `IO80211VirtualInterface` | llw0 |
| Data path | `AppleBCMWLANSkywalk{Tx,Rx}{Submission,Completion}Queue`, `AppleBCMWLANSkywalkPacketPool` | `IOSkywalk{Tx,Rx}{Submission,Completion}Queue`, `IOSkywalkPacketBufferPool` (U) | — |

The most-used imports by symbol count are `IO80211SkywalkInterface` (119), `IO80211BSSBeacon` (111),
`IO80211VirtualInterface` (96), `IO80211InfraInterface` (69) and `IO80211Controller` (64). IO80211Family 25G241 defines
181 metaclasses, among them `WCL*` (WCLJoinManager, WCLRoamManager, WCLScanManager, …: the Family's own connection logic),
`IO80211AWDLPeerManager`, `IO80211P2PSupervisor`, `IO80211P2PDFSProxyManager` and a full NAN stack.

### 2.3 Where AWDL runs on a Mac

- `AppleBCMWLANProximityInterface` implements 137 methods. Among them are 49 AWDL selectors that hand state to the firmware:
  `setAWDL_SYNC_PARAMS`, `setAWDL_SYNCHRONIZATION_CHANNEL_SEQUENCE`, `setAWDL_SYNC_FRAME_TEMPLATE`, `setAWDL_ELECTION_*`,
  `setAWDL_MASTER_CHANNEL`, `setAWDL_PRESENCE_MODE`, `setAWDL_ENCRYPTION_KEYS`, `setAWDL_DFSP_CONFIG`,
  `setAWDL_DFSP_UCSA_CONFIG`, `setAWDL_PEER_TRAFFIC_REGISTRATION`, `setAWDL_OOB_*`, and others. There are also
  `setLLW_PARAMS`, `setPSF_DWELL` and `setP2P_COEX_PROTECTION`. Its strings include `wlan.awdl.params` and
  `wlan.dfsproxy.enabled`.
- So on a Mac, the host (Family + wifip2pd/sharingd) decides *what* the AWDL schedule is, and the **chip firmware executes
  it**: availability windows, channel sequence and sync frames. The infra association keeps running beside it.
- **DFS proxy** (AWDL on a DFS channel) exists: IO80211Family has `IO80211P2PDFSProxyManager` and the string
  `DFS Proxy (AWDL on DFS) is supported on this device`. Its activation strings tie it to AirPlay: `received a UMI with
  request to jumpstart DFSProxy, we are not an ATV !` and `… we are not Airplaying !`. So for AirDrop with the AP on a DFS
  channel, a Mac's AWDL uses a non-DFS channel. The infra link stays on the DFS channel, and the two share the radio in
  time. **Parity target:** AirDrop works while associated on a DFS channel, without disassociation and without a boot-arg.

### 2.4 Bluetooth on a Mac

- Macs use Apple/Broadcom controllers. bluetoothd sends many Broadcom/Apple vendor-specific commands: 328 distinct
  `BD_VSC_*`/`BT_VSC_*` names in the 25G241 binary, for example `BD_VSC_LE_META_ADD_ADV_MATCHING_RULE`,
  `BD_VSC_LE_META_ADV_PCF_SERVICE_UUID`, `BD_VSC_LE_META_ENA_FILTERING_FEATURES`,
  `BD_VSC_OLYMPIC_ADV_DUPLICATE_FILTER_BYPASS_CONFIG` and `BD_VSC_LE_META_ADD_IRK_TO_LIST`.
- For a non-Apple USB controller bluetoothd logs `Found external Bluetooth dongle -- setting chipset to
  THIRD_PARTY_DONGLE`. It reads the NVRAM keys `bluetoothInternalControllerInfo` and `bluetoothExternalDongleFailed`.
  `Unknown chipset %d` is a fault path. (All of these strings are in bluetoothd.)
- The live controller reports `Chipset: THIRD_PARTY_DONGLE`, `Transport: USB`, firmware `v9735 c9236`, and supported
  services `HFP AVRCP A2DP HID Braille LEA AACP GATT SerialPort`.

## 3. How each candidate does it

### 3.1 Structure

| Candidate | Controller / infra / AWDL classes | AWDL timing | Other parts |
|---|---|---|---|
| **MtkWireless 1.1.3** (live) | `AirportMtkWlan : IO80211Controller`, `AirportMtkSkywalkInterface : IO80211InfraProtocol`, `AirportMtkAwdlInterface : IO80211AWDLProtocol` | Host scheduler with ROC + PM handshake, 300 ms off-channel cap (`MtkMwx.cpp:11000–11002`) | HAL from SimpleMtkWlan (OpenBSD `mwx(4)` + net80211), protocol logic in Rust crates, BT as a separate Lilu plugin |
| **AirportMtkWlan 2.3.x** | `AirportItlwm : IO80211Controller`, `AirportItlwmSkywalkInterface : IO80211InfraProtocol`, `MtkAWDLInterface : IO80211AWDLProtocol` | Host scheduler | `mwx(4)` + net80211, itlwm framework, BT folder |
| **WTLWireless 0.15.5** (MT7925, not our chip) | `WTLWirelessNative : IO80211Controller`, `WTLWirelessNativeIf : IO80211InfraProtocol` | none (no AWDL) | mt76 port (ISC), runtime ABI guard checking class sizes per build (`src/native/MtkNativeAbi.hpp`), BT loader plugin |
| mt7925-macos (MT7925) | not surveyed in depth | — | BSD-3-Clause-Clear mt76 port |

All of them follow the AirportItlwm pattern, and it matches Apple's class choice in section 2.2. The architecture
question is therefore not *which Family classes* to subclass. It is how to run AWDL without a firmware AWDL engine, and
what code we may legally build from.

### 3.2 AWDL selector coverage: MtkWireless vs. Apple

MtkWireless implements 28 of Apple's AWDL get/set selectors (151 methods on `AirportMtkAwdlInterface`). Missing compared
with `AppleBCMWLANProximityInterface`: `setAWDL_DFSP_CONFIG`, `setAWDL_DFSP_UCSA_CONFIG`, `setAWDL_ENCRYPTION_KEYS`,
`setAWDL_ENCRYPTION_TYPE`, `setAWDL_ELECTION_ID`, `setAWDL_OOB_SEC_PAYLOAD_REQUEST`, `setAWDL_RTG_PEER_STATS_CONFIG`,
`setAWDL_TXCAL_PERIOD`, `setLLW_PARAMS`, `setCOMPANION_SKYWALK_IF_CREATE`, `setP2P_COEX_PROTECTION`, `setPSF_DWELL`,
`getAWDL_CCA`, `getAWDL_DEV_STATS` and others. Which of these matter for which Continuity feature is an RE question
(section 6, Q3).

### 3.3 Measured on the owner's machine (same AP, ch 52 / 80 MHz, 802.11ax)

| Driver | Down / up (Mbps) | Responsiveness | Tx rate | Clipboard Mac→iPhone | AirDrop |
|---|---|---|---|---|---|
| MtkWireless 1.1.1 | 94.5 / 42.5 | 418 RPM | 867 | works (flaky) | discovery, no transfer |
| AirportMtkWlan 2.3.3 (rejected) | 95.7 / 36.3 | 245 RPM | 432 | **lost** | discovery, no transfer |
| MtkWireless 1.1.3 (live) | 97.9 / 50.7 | 308 RPM | 867 | not yet tested | not yet tested |

1.1.3 30-minute gateway ping: 1800/1800 replies, p50 1.67 ms, p99 3.78 ms, max 46.8 ms (`logs-local/ewifi2/`). Its
counters at t0 show `MTKWiFiRxFillFail = 45412` and `MTKWiFiRxWrongBSS = 26889`, and no MCU timeouts, DMA stalls or
resets. A Tx rate of 432 is half of 867, which suggests one spatial stream on AirportMtkWlan. Not investigated further
(rejected).

### 3.4 Patterns in MtkWireless that a Mac does not need

- `AwdlNeighbors.cpp` (GPL-2.0-or-later) installs a `fe80::/64` route scoped to awdl0 and a static neighbour entry per AWDL
  peer. Its header comment says that without both, TCP SYNs from the iPhone arrived but no reply left the stack. A real
  Mac needs no driver-installed routes, so some peer or data-path signal the Family expects is missing. Our design must find
  it (section 6, Q2), not copy the route hack.
- `CountryLocale.hpp` documents that the firmware reports country `00`. With locale FCC, macOS planned AWDL on US channel
  149 while the AP was on 44 (Italy). A real regulatory domain is therefore an AWDL-correctness requirement as well as a
  legal one. On this machine 1.1.3 reports `MTKWiFiCountryCode = DE` from 802.11d (ioreg).
- The boot-args (`amtk_awdl_offchan`, `amtk_awdl_idle_gap`, `amtk_awdl_infra_bw`, …) are tuning knobs. Under the parity
  rule they are not acceptable in shipped code.

## 4. Root causes of the observed gaps

### 4.1 AirDrop: discovery works, transfer never (high confidence)

Evidence chain, MtkWireless at `424f976`:

1. `kext/Wifi/vendor/SimpleMtkWlan/hal_mwx/MtkMwx.cpp:852–854` (`mwx_chan_is_dfs`, channels 52–64 and 100–144) and
   `:2815–2818`: DFS channels are flagged `IEEE80211_CHAN_DFS | IEEE80211_CHAN_PASSIVE`, "passive scan only, reported as
   DFS to airportd, never AWDL". `awdlChannelAllowed()` refuses them (`AirportMtkAwdlInterface.cpp:661, 3556, 3568,
   3586`).
2. `MtkMwx.cpp:11004–11015`: the boot-arg `amtk_awdl_offchan=0` disables leaving the AP channel.
   `AirportMtkAwdlInterface.cpp:2533–2544`: with it set, "AWDL only uses the AP's channel while associated", and the
   planner backs off when no window falls on it.
3. The live AP is on channel 52 (`logs-local/ewifi*/…airport.txt`), and `amtk_awdl_offchan=0` is in the boot-args.

So AWDL has no usable channel: the AP channel is DFS (refused), and every other channel is forbidden by the boot-arg.
AirDrop discovery is BLE plus AWDL service announcements, and the BLE half works. The transfer needs AWDL data frames and
never starts. This matches the owner's observation on both drivers. AirportMtkWlan was tested with the same AP and its
own DFS handling, which has not been checked. MtkWireless's own docs state the limit: "DFS channels (52–144): never used
for AWDL", and `amtk_awdl_offchan=0`: "AirDrop to phones on other channels fails" (`docs/WIFI.md`).

What a Mac does instead: AWDL runs on a non-DFS channel (44 in ETSI countries, 149 elsewhere) while the infra link stays on
52, and firmware shares the radio between them (section 2.3).

Confirming test, owner's choice (no reboot needed): **T1**, move the AP to a non-DFS channel (36–48) for one test and
send a photo both ways with AirDrop. If it transfers, the cause is confirmed. The driver-side fix is the WIRELESS AWDL
engine (ADR-0001), not a boot-arg. Removing `amtk_awdl_offchan=0` from the legacy kext would be a separate EFI change
(WIRELESS#1). MtkWireless's own warning about it ("unannounced channel switches got the client disassociated") is why it
was set.

### 4.2 Handoff and Universal Clipboard: work, but fail often on every driver (medium confidence, not proven)

Facts:
- The Bluetooth stack was the same in all three tests: MtkWirelessBT 1.1.1 + BlueToolFixup in step 0 and E-WiFi (that PR
  changed only `Kernel/Add[5]`, the Wi-Fi kext), MtkWirelessBT 1.1.3 in E-WiFi2. So "flaky on every driver" fits a cause
  outside the Wi-Fi kext.
- Between 21:30 and 22:20 on this boot, bluetoothd logged `advertisingParametersCb Could not start advertising : STATUS
  712` 11 times, `Advertising failed to start … peripheral manager isn't powered on` 9 times (around the relaunch in
  4.3), and the pair `addScanFilterByUUID failed` + `Failed to add rule` 245 times. The rules it can't add are Apple controller features
  (`BD_VSC_LE_META_ADD_ADV_MATCHING_RULE`, `…_ADV_PCF_SERVICE_UUID`, `…_ENA_FILTERING_FEATURES`). A MediaTek controller
  doesn't implement them.
- Handoff and Clipboard announce themselves over BLE advertising, and Clipboard's payload is fetched peer-to-peer. With
  AWDL blocked (4.1), only transfers that can take the shared LAN would succeed.

Hypotheses: (H-a) dropped or late BLE advertisements and missing controller-side scan filters make the BLE half
unreliable; (H-b) payloads that sharingd/rapportd route over AWDL fail and those routed over the LAN succeed. Test **T3**
(no reboot): 10 Clipboard copies each way with `log stream` on rapportd/sharingd/bluetoothd advertising, recording the
link type and success of each. Mac parity needs the BLE side to look like an Apple controller (ADR-0001, BT shim
research), not just a working Wi-Fi.

### 4.3 iPhone can't be connected over Bluetooth, either side (not root-caused)

Facts:
- AirPods connect and play, so ACL, SSP pairing with accessories and A2DP/AAC work.
- MtkWireless's own table lists "Mac connecting to an iPhone over BLE: untested".
- On this boot, while Bluetooth Settings was open (a `com.apple.BluetoothSettings` discovery session with
  `ClassicScan Pairing` flags), at 21:42:09 bluetoothd lost its stack (`Scan start failed … BT_ERROR_STACK_NOT_READY`,
  `OI_HCI_Reset(), HCI already initialized`, USB pipe aborts on the Bluetooth controller). At 21:42:21 it logged the fault
  **`Unknown chipset 0`** and was relaunched (PID changed at 21:42:22). We don't know if the owner was trying the iPhone
  at that moment.

Hypotheses: (H-c) pairing or connecting to an iPhone goes through a path that resets the controller, and after the reset
bluetoothd takes the internal-chipset path with the zeroed `bluetoothInternalControllerInfo` (chipset 0) and aborts; (H-d)
the iPhone requires a feature the controller or the patch set lacks (LE Secure Connections with CTKD, Apple VSCs for
IRK lists). Test **T2** (no reboot): turn on HCI file traces (MtkWireless `docs/BLUETOOTH.md`, "HCI trace"), try
pairing from the Mac and then from the iPhone, and save the `.pklg` plus a `log show` of that minute. The trace stays
local in the control plane's `.private/` because it contains device addresses.

### 4.4 Continuity Camera: wired only, recognized, no image

- The wired path is USB plus a video stream decoded on the Mac, so Wi-Fi and Bluetooth are not involved. Measured on this
  boot: `VTIsHardwareDecodeSupported` returns true for HEVC (`hvc1`) and false for H.264 (`avc1`). So "no HW HEVC decode"
  isn't the whole explanation: either the decoder Vermilion reports fails, or the stream is H.264. That is Ryzentosh-GPU
  work (RB-0003g, dGPU#5), and this RB hands it over.
- Wireless Continuity Camera uses AWDL and depends on 4.1. Re-test after the AWDL engine works.

## 5. License facts that shape the design

| Source | License (checked) | Use in GPL-3.0-only WIRELESS |
|---|---|---|
| Linux mt76 `mt7921/*.c`, `mt792x_core.c` (v6.18) | ISC (first line of each file) | **copy/port** with notices |
| Linux `drivers/bluetooth/btmtk.c` (v6.18) | ISC | **copy/port** (MediaTek BT firmware download, WMT) |
| Linux `drivers/bluetooth/btusb.c` | GPL-2.0-or-later | port allowed (or-later), only if needed |
| OpenBSD `net80211`, `mwx(4)` | ISC/BSD per file | copy/port |
| WTLWireless | ISC, some ISC AND BSD-3-Clause-Clear | copy/port, keep notices |
| mt7925-macos | BSD-3-Clause-Clear | copy/port, keep notices |
| MtkWireless `rust/mtk-wlan` | GPL-2.0-only (Cargo.toml) | study only |
| MtkWireless `rust/mtk-bt` | GPL-3.0-only | copyable (but Rust, see ADR) |
| MtkWireless `rust/mtk-kernel`, `mtk-common`, `mtk-ffi` | MIT | copyable |
| MtkWireless `kext/Wifi/vendor/AirportMtkWlan/AirportMtkAwdlInterface.*`, `AwdlNeighbors.*`, `CountryLocale.hpp` | GPL-2.0-or-later (SPDX line) | license-compatible, but they include `Airport/macOS26/*` headers and the GPL-2.0-only HAL, so as a unit they are study-only; ideas only |
| MtkWireless `kext/BT/*` | repo says GPL-3.0-only; files have no SPDX; `Patcher.cpp`/`BtLoader.cpp` say "Based on SimpleMtkBtPatcher / SimpleMtkBtFw (zxystd, laobamac)" | **license-auditor must trace the zxystd/laobamac origin before any copy** |
| AirportMtkWlan | GitHub reports GPL-3.0; CREDITS says GPL-3.0-or-later; 261 of 269 source files have no SPDX line | top-level grant over code from SimpleMtkWlan (README: GPLv2) and itlwm (GitHub: GPL-2.0). Copy only files with their own GPL-3.0-compatible header and an owner traceable to AirportMtkWlan |
| SimpleMtkWlan, AirportMtk-AWDL, AirportMtk-macOS | GPLv2 text, mostly no SPDX | study only, except files that say "or later" |
| `Airport/*` reverse-engineered Apple headers carried by all IO80211 projects | origin zxystd/itlwm, license not stated per file | **don't copy**; generate our own from the KDK (ADR-0001, issue W2) |

## 6. Open RE questions (assigned to implementation issues, not run now)

- **Q1 (W2):** exact 25G241 vtable layout and class sizes of `IO80211Controller`, `IO80211InfraProtocol`,
  `IO80211AWDLProtocol`, `IO80211SkywalkInterface` and the Skywalk queues, generated from the KDK with REA, with a
  class-size guard like WTLWireless's.
- **Q2 (W6):** what the Family needs to see for an AWDL peer (peer add events, `setAWDL_PEER_TRAFFIC_REGISTRATION`, data-path
  enable) so that awdl0 IPv6 and neighbour resolution work without driver-installed routes.
- **Q3 (W6):** which missing selectors (3.2) each Continuity feature uses: AWDL encryption for Sidecar/Universal Control,
  LLW for Sidecar, `COMPANION_SKYWALK_IF_CREATE`.
- **Q4 (W8):** what bluetoothd does when an Apple VSC fails on a THIRD_PARTY_DONGLE controller (fallback or loss), and
  which VSCs Continuity advertising needs. Can MediaTek's Microsoft-extension advertisement monitor (enabled by Linux
  btmtk for MT7921/7922) provide the same result?
- **Q5 (W9):** the bluetoothd `Unknown chipset` path and what triggers it after a controller reset.
- **Q6 (later):** whether macOS 26 Continuity features use NAN (`IO80211NAN*` classes are present) anywhere a Mac would.
