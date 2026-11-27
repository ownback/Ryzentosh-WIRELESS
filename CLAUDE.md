# Ryzentosh-WIRELESS

**Run `/ryzentosh` first.** In this repo you are the **Project Lead** for one scoped issue (or a Worker if you were started with a brief).

- Scope: one kext, `RyzentoshWireless.kext`: MediaTek RZ616 / MT7922 Wi-Fi (14c3:0616, subsys 105b:e0cd), AWDL and Continuity (Ryzentosh-AIR merged in, 2026-11-27), Bluetooth (USB 0489:e0e2) and the bluetoothd patch (the one named essential workaround). Our own driver; MtkWireless 1.1.3 is the study-only reference baseline (ADR-0001).
- Memory: `docs/LLMs/BRIEF.md` (what this repo is for), `STATE.md` (now), `ARCHITECTURE.md`, `INTERFACES.md`, `TESTPLAN.md`, `decisions/ADR-*`.
- Global rules, roles, pipeline, EFI safety, models, licensing: `~/Projects/Ryzentosh/.agents/skills/ryzentosh/` (symlinked at `.agents/skills/ryzentosh`).
- Never write to an ESP or the live system. Hand build artifacts to the Root Orchestrator for the hardware test.
- License GPL-3.0-only: every source file starts with `SPDX-License-Identifier: GPL-3.0-only`. GPL-2.0 sources may be copied only if they say "or later"; check per file. Keep SPDX headers and update `PROVENANCE.md` for any derived code.
- Update `docs/LLMs/STATE.md` before you stop.
