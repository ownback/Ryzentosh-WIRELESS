# Ryzentosh-WIRELESS

`RyzentoshWireless.kext`: Wi-Fi, AWDL and Continuity (AirDrop, Handoff, Universal Clipboard, Sidecar, Continuity
Camera), and Bluetooth for the MediaTek RZ616 / MT7922 card (Wi-Fi PCI `14c3:0616`, subsystem `105b:e0cd`; Bluetooth USB
`0489:e0e2`). One GPL-3.0-only kext. Ryzentosh-AIR was merged into this repo on 2026-11-27 and archived.

It also carries the bluetoothd patch (a Lilu plugin), the project's **one named essential workaround**: macOS rejects
non-Apple Bluetooth controllers. The patch is minimal, signature-checked, fail-closed and behind a feature flag (ADR-0001,
[#10](https://github.com/ownback/Ryzentosh-WIRELESS/issues/10)).

When it passes on the reference hardware, it replaces MtkWirelessWifi, MtkWirelessBT and BlueToolFixup in the EFI. It is
our own driver. MtkWireless 1.1.3 is only the reference baseline to beat: it is studied, never copied
([ADR-0001](docs/LLMs/decisions/ADR-0001-ryzentoshwireless-architecture.md)).

## Where it fits
One Ryzentosh repo ships one kext, named after the repo. The target EFI is described in
[EFI-END-STATE.md](https://github.com/ownback/Ryzentosh/blob/main/docs/Architecture/EFI-END-STATE.md).

| Repo | Ships | Contains |
|---|---|---|
| Ryzentosh-WIRELESS (this repo) | RyzentoshWireless.kext | Wi-Fi, Bluetooth, AWDL/Continuity, bluetoothd patch |
| Ryzentosh-GPU (was Ryzentosh-dGPU; Ryzentosh-iGPU archived) | RyzentoshGPU.kext | RX 7900 XTX and the Raphael iGPU |
| Ryzentosh-CPU | RyzentoshCPU.kext | CPU PM and sensors, the AppleMCEReporter blocker, the AMD-V backend for Hypervisor.framework |
| Ryzentosh-PLATFORM | RyzentoshPlatform.kext | NVMe power management, the USB map, what RestrictEvents does today |
| Ryzentosh-ACPI | SSDTs | every SSDT, one per purpose |
| Ryzentosh-EFI | the OpenCore EFI | config, kext list |

Part of [Ryzentosh](https://github.com/ownback/Ryzentosh): macOS Tahoe on an AMD PC, built to behave like a Mac Apple designed and
supports. Written by AI agents. Apple is not involved in this project.

**Status:** research. Nothing here is ready to install yet. See [docs/Humans](docs/Humans/) and the
[compatibility matrix](https://github.com/ownback/Ryzentosh/blob/main/docs/Hardware/COMPATIBILITY_MATRIX.md).

**Releases:** RyzentoshWireless.kext (one binary).

## Reference hardware
AMD Ryzen 9 7900X3D · Radeon RX 7900 XTX (XFX) · ASUS ROG STRIX B650E-I · macOS 26.7.1 (25G241) · OpenCore 1.0.8.

## Contributing
See [CONTRIBUTING.md](https://github.com/ownback/Ryzentosh/blob/main/CONTRIBUTING.md). AI agents: read [AGENTS.md](AGENTS.md).

License: GPL-3.0-only (relicensed from GPL-2.0-only on 2026-11-27, before any code existed). The hardware layer is ported from ISC-licensed sources (Linux mt76, btmtk) and OpenBSD net80211; GPL-2.0-only sources such as MtkWirelessWifi are study-only. The licence plan is in ADR-0001 §4. Third-party code and its origins are listed in [PROVENANCE.md](PROVENANCE.md).
