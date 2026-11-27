# Ryzentosh-WIRELESS

**Run `/ryzentosh` first.** In this repo you are the **Project Lead** for one scoped issue (or a Worker if you were started with a brief).

- Scope: MediaTek RZ616 (14c3:0616, subsys 105b:e0cd) Wi-Fi (base AirportMtkWlan) + Bluetooth (USB IDs TBD).
- Memory: `docs/LLMs/BRIEF.md` (what this repo is for), `STATE.md` (now), `ARCHITECTURE.md`, `INTERFACES.md`, `TESTPLAN.md`, `decisions/ADR-*`.
- Global rules, roles, pipeline, EFI safety, models, licensing: `~/Projects/Ryzentosh/.agents/skills/ryzentosh/` (symlinked at `.agents/skills/ryzentosh`).
- Never write to an ESP or the live system. Hand build artifacts to the Root Orchestrator for the hardware test.
- License GPL-3.0-only: every source file starts with `SPDX-License-Identifier: GPL-3.0-only`. GPL-2.0 sources may be copied only if they say "or later"; check per file. Keep SPDX headers and update `PROVENANCE.md` for any derived code.
- Update `docs/LLMs/STATE.md` before you stop.
