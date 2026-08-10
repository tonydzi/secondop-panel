# PROMPT.md — put a panel of breakers on your own pipeline in ~15 minutes

Copy everything below the line into **Claude Code, Codex, Cursor or any coding agent** working in your repo. It installs the panel and wires it into the three places where a second opinion actually changes the outcome.

---

I want a **panel of adversarial reviewers from different model families** on my pipeline, not a single "ask another model" call. Do this end to end and report what you actually verified.

**Step 1 — install.**
Clone `https://github.com/tonydzi/secondop-panel` into a tools directory (or vendor `secondop_panel.py` — it is one stdlib-only file, Python 3.8+, no dependencies). Run `python secondop_panel.py selftest`. It must print 30 checks, 0 failed. If it does not, stop and tell me why.

**Step 2 — pick my rails.**
Read `docs/MODEL-FAMILIES.md`, then look at what this machine actually has: which model CLIs are on PATH and logged in, and which API keys exist in the environment. Propose **3–4 rails from 3–4 DIFFERENT labs** and tell me the shortfall honestly if I only have two. Write `panel.json` from `panel.example.json`, delete the rails I do not have, and set `family` to the lab for each one.

Then run the honesty probe on every candidate — this is the selection criterion, not a formality:

```
python secondop_panel.py t3 --paths-only --files <some real file> --rails <one rail>
```

A rail that answers *"I cannot see the code"* passes. A rail that returns confident findings about a file it was never shown **fails** — report it to me and suggest a replacement, because that model will fabricate whenever my prompt is incomplete.

Finish with `python secondop_panel.py doctor` and show me the table: `OK` / `DEGRADED` / `ABSENT` per rail, and whether the live families meet quorum.

**Step 3 — wire in the three touchpoints.**
Add these to whatever ritual or script I use before saying work is done (a Makefile target, a git hook, a CI job, or your own checklist — ask me which if it is not obvious from the repo):

* `t1` **before writing code** on any substantive change: `python secondop_panel.py t1 --task <id> --context "<the plan, in full>"`
* `t2` **at a real fork** (two designs on the table): `python secondop_panel.py t2 --task <id> --context "<option A / option B / which I lean to and why>"`
* `t3` **after the build**, always with file contents: `python secondop_panel.py t3 --task <id> --files <every changed file> --context "<what changed and why>" --diff HEAD~1`

**Step 4 — the two rules that make it worth having.** Implement both, do not just describe them:

1. **Contents, never paths.** Pass real files to `--files` so the packer sends line-numbered contents. Never hand the panel a list of file names and never paste a summary in place of the code — measured on this tool's own repo, a paths-only panel produced 20 findings and 0 of them were about anything that exists in the file, while all four models sounded completely certain.
2. **Exit code 3 downgrades my verdict.** If the panel exits 3, the ritual must report a **warning**, never a pass, and must name which families were missing. A skipped rail that silently costs nothing is a gate nobody reads. Show me the exact lines you changed to enforce this.

**Step 5 — prove it works, on this repo, out loud.**
Run `t3` against your own most recent change with real file contents. Show me: the rails that answered, the rails that did not **and why**, the distinct family count, the exit code, and the findings. Then tell me which findings are real, which are by-design behaviour and which are speculation — the panel produces candidates, you arbitrate, I decide. Do not describe anything you did not run.

Constraints: do not put API keys in any file (environment variables only, `key_env` names them); do not let a failing rail fall back to another vendor (unknown or dead rail = skip, that is the whole point); do not add a second rail from a lab that is already in the panel and call it independence.
