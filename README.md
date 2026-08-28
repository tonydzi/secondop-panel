# secondop-panel

**Nobody reviews themselves.** The usual fix — "have another model check it" — quietly collapses into asking *one* other model, and one model is one opinion with one blind spot.

This is the panel: a review request fanned out to **several model families at once**, with a verdict contract, quorum counted by family, and an exit code that refuses to go green when there was no real second opinion.

Stdlib-only Python, one file, MIT. Works with local CLIs (`codex`, `grok`, `gemini`, `claude`), any OpenAI-compatible HTTP endpoint (OpenRouter, vendor APIs, your own gateway), or a mix.

Built and used daily at [Palo Alto AI Research Lab](https://github.com/tonydzi/tonydzi) — it is the gate our agents must pass before they are allowed to say "done".

---

## The one thing to take away: send CONTENTS, not paths

If you hand a panel a list of **file paths**, the models that cannot read your disk will not tell you. They will invent.

We measured it on this very repository, 2026-08-10, four models from four different families, same code, same reviewer role — only the material changed:

| what the panel was given | rails that answered | findings | findings anchored on something that actually exists | models that said "I cannot see the code" |
|---|---|---|---|---|
| **paths only**, framed the way an integrator naturally writes it | 4 / 4 | 20 | **0** | **0 / 4** |
| **paths only**, but the prompt says contents are missing and honesty is allowed | 2 / 4 | 0 | — | **2 / 2** |
| **full file contents** | 2 / 4 | 10 | **10** | — |

The paths-only run was not "weaker". It was *confidently wrong*: findings about a `families_needed` counter, a `--model-config` CLI flag, per-request subprocess spawning and unlocked concurrent log writes. None of those exist here — the file has a two-level lock and no such flag. All four models produced a clean, numbered, professional-looking review of a program they had never seen.

The contents run named real functions — `is_finding`, `families_answered`, `_run_cmd_rail`, `strip_prefixes` — and **four of its findings were real defects, fixed in this repo before publishing** (see [GOTCHAS](docs/GOTCHAS.md#found-by-the-panel-on-its-own-code)).

So two rules are wired into the tool, not left to your discipline:

1. `pack_files()` sends **line-numbered file contents**; missing and truncated files are named explicitly, because a silent absence invites invention while a named absence does not.
2. `--paths-only` exists so you can reproduce the failure yourself — and a run in that mode **can never exit 0**.

And a third rule for choosing models: **an honest "I don't have the data" beats a confident guess.** That is a selection criterion, not a nicety — a model that will not say "I can't see it" is a model that will fabricate under pressure. Test yours: `python secondop_panel.py t3 --paths-only --files anything.py`.

---

## Three touchpoints, not "review at the end"

| point | when | question | role |
|---|---|---|---|
| `t1` | before you write code | "is this plan sound?" | `verify` |
| `t2` | at a fork | "which of these two designs?" | `counter` |
| `t3` | after the build | "try to **break** what I just made" | `break` |

A panel called only at `t3` finds bugs. A panel called at `t1` prevents the build.

```bash
python secondop_panel.py t1 --task new-cache --context "the plan, in full"
python secondop_panel.py t2 --task new-cache --context "option A ... option B ... I lean A because ..."
python secondop_panel.py t3 --task new-cache --files src/cache.py tests/test_cache.py \
       --context "what I changed and why" --diff HEAD~1
```

---

## Quickstart

```bash
git clone https://github.com/tonydzi/secondop-panel.git && cd secondop-panel
cp panel.example.json panel.json         # edit: keep the rails you actually have
export OPENROUTER_API_KEY=...            # or point the rails at your own endpoints
python secondop_panel.py selftest        # offline: 32 checks, no network, no keys
python secondop_panel.py doctor          # which rails are alive on THIS machine
python secondop_panel.py t3 --task my-fix --files src/a.py --context "what I did"
```

Exit codes are the gate, not decoration:

| code | meaning | what your ritual should do |
|---|---|---|
| `0` | quorum met — N distinct **families** answered | a pass is allowed |
| `3` | no independent second opinion (rails dead, skipped, all one family, or a degraded paths-only run) | report at most a **warning**, never a pass |
| `2` | bad config / nothing to review | fix and re-run |

---

## What makes it a panel and not a lucky loop

**Quorum counts FAMILIES.** `google/gemini-*` through a gateway and the local `gemini` CLI are two *doors to one lab*, not two opinions. Counting heads instead of families is how a tool reports "four reviewers agreed" after asking the same lab four times. The threshold never learns vendor names — add or remove rails freely, the rule does not change.

**No silent vendor substitution.** An unknown rail name is a **skip**, never a redirect into whichever rail happens to work. A verdict from vendor A printed under the name of vendor C makes the independence fake and your gate green on one opinion. Duplicates (`--rails codex,codex`) are dropped before the run for the same reason.

**A skip is visible and it costs you the pass.** Rails that were asked and did not answer are printed with the reason, and the run cannot return 0 unless enough distinct families spoke. "Two rails" while one is alive is a lie your CI will believe.

**A closed door is not a dead vendor.** `doctor` separates `ABSENT` (this machine never had it) from `DEGRADED` (installed and refusing) — different diseases, different cures. A vendor is only dead when *every* door is closed; the browser is usually still open.

**One rail may never take the panel down.** A vendor that crashes is a missing opinion, not the loss of everyone else's.

---

## Configuration

`panel.json` — rails are just data (see [`panel.example.json`](panel.example.json)):

```json
{ "quorum": 2, "timeout_s": 180,
  "rails": [
    {"name":"gemini","family":"google","kind":"http",
     "api":"https://openrouter.ai/api/v1/chat/completions",
     "key_env":"OPENROUTER_API_KEY","model":"google/gemini-2.5-flash-lite","effort":"high"},
    {"name":"grok","family":"x-ai","kind":"cmd","unset_env":["XAI_API_KEY"],
     "cmd":["grok","--prompt-file","{prompt_file}","--tools","","--no-plan"]}
  ] }
```

`{prompt_file}` is replaced with a temp file holding the prompt; rails without it get the prompt on stdin. **Never pass a multi-line prompt as an argv string** — shell/CRT quoting mangles it and the vendor silently sees only the first line.

Keys come from the environment only. Nothing here stores a secret; see [`docs/SECURITY.md`](docs/SECURITY.md) — you are shipping your source code to third parties, and that deserves one page of thought.

Picking the 3–4 families: [`docs/MODEL-FAMILIES.md`](docs/MODEL-FAMILIES.md).

---

## Wiring it into your own agent

[`PROMPT.md`](PROMPT.md) is a copy-paste prompt for Claude Code / Codex / Cursor: *"install a panel of breakers on my pipeline"*, ~15 minutes, including where to put the three calls and how to make a failed quorum downgrade your verdict instead of being ignored.

---

## Every trap we paid for

[`docs/GOTCHAS.md`](docs/GOTCHAS.md) — 19 of them, each one a bug a live panel found in this code: markdown-bolded verdict tags, `VERIFYING` scored as a verdict, `ACCEPT` on line one with `COUNTER` in the body, empty strings counted as opinions, `O_APPEND` not being atomic on Windows, and the reasoning-model trap where a tight token budget returns empty `content` with `finish_reason: stop` so your client invents a silence that never happened.

## License

MIT.

---

Part of the reliability series by Palo Alto AI Research Lab. Closest neighbour: [`verdict-contract`](https://github.com/tonydzi/verdict-contract) — the structured verdict a reviewer must answer in, and the parser that refuses to guess (this kit ships the same discipline for a whole panel). See also [`verified-ops-starter`](https://github.com/tonydzi/verified-ops-starter) (your job says exit 0 — prove it did the work), [`agent-approval-gate`](https://github.com/tonydzi/agent-approval-gate) (when the panel says BLOCK and a human has to decide) and [`telegram-mcp-kit`](https://github.com/tonydzi/telegram-mcp-kit). Questions / broken step? Open an issue — we answer within 24h.

---

> **Publishing your own internals?** This repo was sanitized for release with
> [`oss-publish`](https://github.com/tonydzi/oss-publish) — personal data replaced by plausible
> fakes of the same shape (never `<REDACTED>`), and a fail-closed gate re-scans the whole tree
> before the push. Free, MIT.

<!--kits-series:start-->

## 🧰 Connector & Ops Kits

Eight kits, all published 2026-08-10, each lifted out of the same live fleet after it
survived production rather than written as a demo. They are independent: take one, ignore
the rest. All stdlib-only Python, all free.

| kit | what it solves |
|---|---|
| [`telegram-mcp-kit`](https://github.com/tonydzi/telegram-mcp-kit) | Connect your agent to your own Telegram account in ~15 minutes, with the production patches and every gotcha |
| [`whatsapp-mcp-kit`](https://github.com/tonydzi/whatsapp-mcp-kit) | Link WhatsApp, using a live self-refreshing QR page that makes pairing actually work |
| [`mcp-daemon-diet`](https://github.com/tonydzi/mcp-daemon-diet) | One shared MCP daemon per machine instead of a stdio copy in every session, with a watchdog that will not blind your live sessions |
| [`agent-approval-gate`](https://github.com/tonydzi/agent-approval-gate) | Your agent needs a human's OK and nobody is at the terminal: the ask goes to a messenger, the answer comes back into the run |
| [`fleet-deploy`](https://github.com/tonydzi/fleet-deploy) | Roll a fix to N machines and prove it landed on each one: canary waves and a verify that must read a fact back |
| [`secondop-panel`](https://github.com/tonydzi/secondop-panel) | Nobody reviews themselves, and one reviewer model is one blind spot: fan a change out to several model families with quorum and honest skips |
| [`oss-publish`](https://github.com/tonydzi/oss-publish) | Open up internal work without leaking it: plausible substitutions of the same shape, then a fail-closed gate over the whole tree |
| [`llm-spend-audit`](https://github.com/tonydzi/llm-spend-audit) | What your own wiring charges on every session, and which paid subscriptions are going undrawn |

<!--kits-series:end-->

<!--ecosystem-map:start-->

## 🧩 One piece of a working system

This repository is one piece lifted out of a live operation: one non-technical founder, an AI
cofounder, and a fleet of machines that reach consensus with each other and wake the human only
for money or the irreversible. It was extracted after it survived production, not written as a
demo — and it runs on its own: nothing here phones home to the rest.

**See how the whole thing fits together → [SYSTEM.md](https://github.com/tonydzi/tonydzi/blob/main/SYSTEM.md)**

<!--ecosystem-map:end-->

## AI contributors

This project is built by a human + AI team, and the git log says so: Claude writes most of
the code, Codex and Grok review it, Gemini feeds the research. Each is credited on a commit
**only if its output changed that commit's content** — no decorative credits. Lab-wide
policy, one source for every repo: [AI-CONTRIBUTORS.md](https://github.com/tonydzi/.github/blob/main/AI-CONTRIBUTORS.md).
