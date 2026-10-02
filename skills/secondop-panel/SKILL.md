---
name: secondop-panel
description: "Fan a plan, design fork or finished change out to a panel of reviewer models from several different labs (codex, grok, gemini, claude CLIs or any OpenAI-compatible endpoint) with quorum counted by model family. Use before saying work is done, before writing code on a substantive change (t1), at a design fork (t2), or after a build to try to break it (t3). Sends line-numbered file contents, never paths; exit 3 means no independent second opinion."
license: MIT
---

# secondop-panel

Nobody reviews themselves, and one reviewer model is one blind spot. This skill runs `secondop_panel.py` (one stdlib-only Python file in this repo) to ask several model families at once.

## When to call it

| point | when | question | role |
|---|---|---|---|
| `t1` | before writing code | is this plan sound? | verify |
| `t2` | at a fork | which of these two designs? | counter |
| `t3` | after the build | try to break what I just made | break |

```bash
python secondop_panel.py t1 --task <id> --context "<the plan, in full>"
python secondop_panel.py t2 --task <id> --context "<option A / option B / which I lean to and why>"
python secondop_panel.py t3 --task <id> --files <every changed file> --context "<what changed and why>" --diff HEAD~1
```

## Setup (once per machine)

1. `cp panel.example.json panel.json`, keep only the rails this machine really has, set `family` to the lab of each rail. Keys live in environment variables only (`key_env`).
2. `python secondop_panel.py selftest` (offline, 32 checks, must be 0 failed).
3. `python secondop_panel.py doctor` shows each rail as `OK`, `DEGRADED` or `ABSENT`, and whether live families meet quorum.
4. Honesty probe per rail: `python secondop_panel.py t3 --paths-only --files <real file> --rails <one rail>`. A rail that says it cannot see the code passes; a rail that returns confident findings fails and should be replaced.

## Rules

- **Contents, never paths.** Pass real files to `--files`. Measured on this repo: a paths-only panel gave 20 findings, 0 about anything that exists, and no model admitted it could not see the code.
- **Exit codes are the gate.** `0` quorum of distinct families met, a pass is allowed. `3` no independent second opinion (rails dead, skipped, one family, or paths-only): report a warning at most, never a pass, and name the missing families. `2` bad config or nothing to review.
- **Quorum counts families, not heads.** A gateway model and the same lab's CLI are one opinion.
- **No silent vendor substitution.** An unknown or dead rail is a skip, never a fallback to another vendor.
- **The panel produces candidates; you arbitrate.** Sort findings into real, by-design and speculation before reporting, and never describe a run you did not do.
- Running the panel ships your source to third parties: read `docs/SECURITY.md`.

More: `README.md`, `PROMPT.md`, `docs/GOTCHAS.md`, `docs/MODEL-FAMILIES.md`.

<!--kit-footer-->

---

**Like this skill?** It is one of 100 in [second-brain-starter-kit](https://github.com/tonydzi/second-brain-starter-kit): the second brain we built for ourselves and run every day at Palo Alto AI Research Lab. Install the whole set with `npx skills add tonydzi/second-brain-starter-kit`. Everything is open source and free, so take what you need.

Flagships worth a look on their own: [secondop-panel](https://github.com/tonydzi/secondop-panel) (a second opinion from a panel of external models), [claude-memory-tidy](https://github.com/tonydzi/claude-memory-tidy) (stop your agent's memory from rotting), [telegram-mcp-kit](https://github.com/tonydzi/telegram-mcp-kit) (your own Telegram over MCP in about 15 minutes).

Author: **Anton Dziatkovskii**, Palo Alto AI Research Lab. Telegram [@tonydzi](https://t.me/tonydzi) - WhatsApp [+1 341 222 9178](https://wa.me/[id]) - X [[аккаунт]](https://x.com/Tony_Stef_)

**Engineers: want to test-drive this setup?** Message me. I hand out free starter seeds to engineers who test and report back, and custom skill requests are welcome.
