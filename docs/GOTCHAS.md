# Gotchas

Every item below cost us something real. Most were found by a panel reading its own source — which is the point of the tool.

---

## 1. Paths instead of contents → confident fiction ⭐

**The big one.** Give the panel a list of file names and the models that cannot read your disk will not say so.

Measured 2026-08-10, four families, same reviewer role, same files, only the material changed:

* **paths only, natural framing** — 4/4 answered, **20 findings, 0 of them about anything that exists**, and **0/4** models mentioned that they had no code. They invented a `families_needed` counter with a race condition, a `--model-config` CLI flag, per-request subprocess spawning, and unlocked concurrent log writes (the file has a two-level lock).
* **full contents** — every finding named a real function; four of them were real defects (§ *Found by the panel on its own code*).

Fix, wired into the tool rather than left to discipline: contents are line-numbered and sent; missing/truncated files are **named**, never dropped; `--paths-only` is a reproduction mode that can never exit 0.

**Selection criterion that falls out of this:** prefer models that will say *"the file contents were not provided, analysis is impossible"*. With the honesty clause in the system prompt, two of our four said exactly that; without it, none did. A model that cannot admit missing data will fabricate under pressure, and a reviewer that fabricates is worse than no reviewer, because it consumes the attention you would have spent looking yourself.

## 2. "Second opinion" that is one vendor twice

`--rails codex,codex` used to report two answers and exit 0. Duplicates are now dropped before the run, with a warning.

## 3. Quorum by model name instead of family

`google/gemini-*` via a gateway and the local `gemini` CLI are two doors to **one lab**. Counting heads produces "four reviewers agreed" after asking the same lab four times. Quorum counts distinct `family` values; the threshold never learns vendor names.

## 4. Silent vendor substitution

An unknown rail name used to fall through to whichever rail worked. That prints vendor A's verdict under vendor C's name: the independence is fake and the gate goes green on one opinion. Unknown rail = **skip**, always.

## 5. `ABSENT` and `DEGRADED` are different diseases

A machine that never had a vendor is a machine without that vendor. A vendor that **is** installed and still refuses to answer is a broken thing pretending to be a rail. We once had a node where the binary was present but device-auth was never done: it looked equipped and could review nothing. `doctor` separates them.

Corollary: **one closed door is not a dead vendor.** We declared a rail dead for having no CLI while the same vendor was logged in and answering in a browser tab three centimetres away.

## 6. Markdown-decorated verdict tags

Vendors bold the tag: `**VERIFY**`, `` `BLOCK` ``, `## ACCEPT`. A raw `startswith()` scores a real objection as "unknown" — a finding silently lost. Strip decoration first.

## 7. Prefix matching scores non-verdicts

`VERIFYING the empty input` and `NOTVERIFY` both start with `VERIFY`. Anchor on the whole first token (`re.match(r"[A-Z]+")` + exact membership), never a prefix.

## 8. `ACCEPT` on line 1, `COUNTER` in the body

The vendor contradicted itself, and the dangerous direction is the false ACCEPT. This resolves to `None` (unknown → a human looks), never to a silent "agreed".

## 9. An empty string is not an opinion

`if reply is not None` counted `""` as an answer and could hand out a green exit with zero real verdicts. Truthiness on the *stripped* string, everywhere.

## 10. Reasoning models: empty `content`, `finish_reason: stop`

On reasoning models the token budget is consumed by `reasoning` **before** `content`. With a tight `max_tokens` you get a perfectly successful HTTP 200 with an empty answer — and a naive client records "the vendor did not answer". That is a false silence invented by your own code. Keep `http_max_tokens` generous (4000+), and report *"empty content, non-empty reasoning — raise max_tokens"* as its own diagnosis. We hit this live twice in the runs quoted above.

## 11. A non-zero exit must never yield a verdict

A CLI that fails and still prints something would otherwise be parsed as `ACCEPT` and green-light a broken gate. Non-zero exit → no verdict, full stop.

## 12. Multi-line prompts through argv get mangled

Shell/CRT quoting eats them and the vendor sees only the first line, then reviews a fragment with total confidence. Prompts go through a file (`{prompt_file}`) or stdin.

## 13. `O_APPEND` is not atomic on Windows

The CRT does seek-to-end and write as two steps; a 20-thread test lost records. Two live sessions running the gate at once is the real case. The log takes both a `threading.Lock` and an OS advisory lock (`msvcrt.locking` / `flock`) — and if it still cannot write, it says so on stderr instead of losing the record quietly.

## 14. Truncation and skips must be as loud as findings

A silent truncation reads afterwards as "the whole file was reviewed". Any file cut at the byte budget, or unreadable, is printed before the verdicts.

## 15. Quorum built on essays, not verdicts

The contract lived in the prompt but not in the counting: any non-empty reply was an opinion. Two rails answering *"Looks fine overall."* with no verdict tag produced two families, zero findings and **exit 0** — "two families said words" reported as "two families judged". Quorum now counts only machine-countable verdicts; off-contract replies are printed and named, and they do not buy the pass.

## 16. Files dropped for prompt budget were invisible to the operator

The prompt told the *models* that files were not sent; the stats never told the *human*, so the "not fully seen" warning never fired and a partial pack could still exit 0. A drop the caller cannot see is a silent truncation with extra steps.

## 17. A hung rail vanished from the denominator

A rail whose thread never returned was simply absent from the results, so a panel of three printed "2/2 rails answered" and the hung family was missing from the missing list. A panel that quietly shrinks while reporting a full house is the exact failure this tool exists to prevent.

## 18. `--diff` failures were swallowed

A bad ref or a non-repo directory returned `""` with no message, and the panel reviewed without the diff you explicitly asked for — then passed. Missing material must always be audible: it now warns, and an empty diff says so too.

## 19. The selftest ran the branch nobody sees

Every panel check in the selftest called `run_panel(..., quiet=True)` — so the *printing* branch was never executed, and a `TypeError` lived there through a completely green suite. It was in the one message that must never fail: the gate's own *"no independent second opinion"* report. The panel exited **1** (crash) instead of **3**, which a caller reads as "the tool is broken" rather than "the review did not happen" — the failure hiding inside the failure handler.

It was found by running the published tool on purpose with a dead rail, not by the tests. So: **exercise the human-facing output in the test**, and after publishing, clone your own repo into a clean directory and break it on purpose. What you tested is not what you shipped until you have run what you shipped.


## The tests must be able to fail

Every rule above has a check in `selftest`, and the checks are verified by mutation: break the exact line the rule protects and the selftest must redden. Seven mutations, seven kills — exact-token matching, family-based quorum, the contract check, the dropped-file counter, the hung-rail backfill, config validation, and the empty-reply guard. A green test suite that stays green on broken code is not evidence, it is decoration. The check count in the summary line is derived from the checks that ran, never typed in — a hard-coded total drifts the moment someone adds a check, and then a tool about honesty ships a small lie in its own output.

---

## Found by the panel on its own code

The `t3` run against this repository (2026-08-10, contents sent) produced ten findings anchored on real symbols. Four were genuine and are fixed here:

| finding | rail | fix |
|---|---|---|
| Two rails missing both `name` and `family` collapsed into the family `"?"` — a panel of two vendors then counted as one opinion, and the exit-3 reason was invisible | Gemini | `validate_conf()` refuses nameless/familyless rails at the door |
| `strip_prefixes` removed only **one** banner line, so a rail printing two banners left the verdict tag on line 2 and the verdict was lost | DeepSeek | strip repeatedly (bounded) until the tag is first |
| `log_usage` gave up after 60 retries and dropped the record **silently** — on a read-only dir the counter goes quiet exactly when you are watching it | DeepSeek | a `WARN` on stderr; the review itself is never killed over a log |
| File truncation was recorded in the packer stats but never shown to the human | Gemini | printed before the verdicts (gotcha 14) |

The other six were by-design behaviour (the `ACCEPT`+`COUNTER` → unknown rule) or speculation. That ratio is normal and is the reason a panel needs a human to arbitrate: **the panel finds candidates, you decide.**

One more, found by simply running `doctor`: it probed **disabled** rails and counted them toward quorum, so it reported "quorum OK" on a machine where the panel itself would have exited 3. A health check that answers a different question than the gate is worse than no health check.

### Round two — the final pre-publish run

The fixed file went back to the panel with a fifth rail added (a local CLI, i.e. different plumbing as well as a different lab). It found four more, all real, all fixed above: gotchas **15–18**. The rail that found them was the one that was *not* behind the shared gateway.

Two lessons from that, both cheap to copy:

* **Re-run the panel after you fix what the panel found.** The second round is not a formality — it produced the sharpest list of the day, because the easy findings were gone and the reviewers had to look at the real seams.
* **Keep at least one rail on different plumbing.** Three of five rails went through one gateway; when that gateway's reasoning models ate their token budget (gotcha 10), the CLI rail was the panel.

In the same run, three of the five "findings" from the gateway rails were about behaviour that had already been fixed or was documented as intentional. That is the normal ratio, and it is why the panel proposes and a human disposes.
