# BRIEF: Ryzentosh-WIRELESS

**Scope:** one kext, `RyzentoshWireless.kext` (GPL-3.0-only): MediaTek RZ616 / MT7922 Wi-Fi (PCI 14c3:0616, subsys 105b:e0cd), AWDL and the Continuity features (Ryzentosh-AIR merged in, 2026-11-27), Bluetooth (USB 0489:e0e2) and the bluetoothd patch (the named essential workaround). Architecture and sources: ADR-0001 (own driver from ISC/BSD primaries; MtkWireless 1.1.3 is the reference baseline, study-only).

**Release artifact:** RyzentoshWireless.kext (one binary).

**Depends on:** [Ryzentosh-EFI]

**Parity targets:** see the rows for this project in the control plane's COMPATIBILITY_MATRIX.md.

**Out of scope:** anything another Ryzentosh repo owns (see projects.yaml).
