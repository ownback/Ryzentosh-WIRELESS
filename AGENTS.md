# For AI agents (Claude Code, Codex, others)

Load the skill at `.agents/skills/ryzentosh/SKILL.md (symlink to the control plane; if it's missing, see https://github.com/ownback/Ryzentosh/tree/main/.agents/skills/ryzentosh)` before doing anything (Claude Code: `/ryzentosh`). The short version:

1. **Parity rule.** If a real Mac can't do it, Ryzentosh doesn't. If a real Mac can, Ryzentosh must, in full. Classify bugs (macOS / driver / config) only with evidence.
2. **No workarounds** in shipped code: no post-login scripts, replayed modes, timing sleeps, polling daemons or boot-arg crutches.
3. **Engineering bar:** Apple's own driver teams. Read `references/apple-driver-quality.md`. Never claim Apple approval.
4. **Never write to an ESP, `/System` or `/Library/Extensions`** unless you are the Root Orchestrator holding the EFI lock. The internal ESP is shared with Linux.
5. **Licensing:** read a LICENSE before using code. No license means study only. Record provenance for every derived file.
6. **Privacy:** never commit serials, SMBIOS identity, MAC addresses, credentials, home paths or personal data.
7. **Pipeline:** investigate → reason → plan → tests → code → refactor → build → independent review → real-hardware test.
8. **Write back** what you learned to `docs/LLMs/STATE.md` before you stop.

External contributors' agents: open an issue first for anything larger than a small fix. PRs must include test evidence.
