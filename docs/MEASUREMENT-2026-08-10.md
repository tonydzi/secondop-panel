# Measurement, 2026-08-10 — paths vs contents

Raw output of the three runs quoted in the README. Same four rails (four different labs, via one
OpenAI-compatible gateway), same reviewer role (`break`), same two files under review
(`secondop_panel.py`, `panel.example.json`). **Only the material changed.**

Verbatim, unedited except for the removal of nothing — this is what came back.

---

## Run A1 — paths only, natural framing (the naive baseline)

Prompt: an adversarial-reviewer instruction, a rich description of what the tool does, and
`Changed files: secondop_panel.py / panel.example.json`. Nothing signalled that the contents were
absent — exactly how an integrator writes it the first time.

**Result: 4/4 answered, 20 findings, 0 anchored on anything that exists, 0 models mentioned the
missing code.**

```

[PANEL exp-A1b-naive-rich | GLM | z-ai/glm-4.7-flash]
COUNTER

1.  **Fragile Verdict Parsing**: LLMs typically append preamble text (e.g., "I think the verdict is accept") rather than outputting a single token as the start of the string, causing the tool to treat every valid review as a parsing failure.
2.  **Race Condition in Quorum Counting**: Concurrent threads incrementing the `families_needed` counter without a lock can result in simultaneous decrements (e.g., two threads running at the exact same tick), causing the consensus calculation to go negative and fail when it should succeed.
3.  **Exit Code Drift on Parse Error**: An unhandled exception during the parsing of a successful HTTP response (e.g., missing a field in the JSONL log) propagates to the main process and causes it to exit with code 1 (Python default) instead of code 3, falsely signaling a CI pass when no second opinion was gathered.

[PANEL exp-A1b-naive-rich | GEMINI | google/gemini-2.5-flash-lite]
COUNTER

The work described has several potentially damaging concrete defects. The most critical ones revolve around the core verdict contract enforcement and the quorum counting mechanism, which can be exploited to allow malicious code through CI gates or to create denial-of-service conditions.

Here are the most damaging defects:

1.  **Verdict Misinterpretation leading to Malicious Code Acceptance:** The reliance on the "first token" for verdict contract enforcement is brittle. Models can be prompted or may naturally output variations (e.g., `ACCEPTABLE`, `ACCEPTING`, `ACCEPT:`) that are not exact matches. If the tool's parsing logic is too lenient (e.g., `token.startswith('ACCEPT')`) or incorrect, it can misinterpret a `BLOCK` or `COUNTER` verdict as `ACCEPT` for malicious code.
    *   **One-line repro:** `secondop-panel review --code '<malicious_code_designed_to_output_ACCEPTABLE_or_ACCEPT_with_caveats>' --model-config '[{"family": "vulnerable_family", "endpoint": "http://compromised-model.com"}]'`

2.  **Exploitable Quorum Dominance by Biased Model Families:** The quorum is counted by model *family*. If one family contains many models (e.g., multiple OpenAI-compatible endpoints) and is known to be biased towards `ACCEPT`, it can artificially create a quorum favoring acceptance for malicious code, overriding cautious opinions from other families.
    *   **One-line repro:** `secondop-panel review --code '<malicious_code>' --model-config '[{"family": "always_accept_family", "endpoint": "..."}]' * N` (where N is large enough to form a quorum, and `always_accept_family` consistently returns `ACCEPT`).

3.  **Race Condition in Verdict Aggregation:** With parallel execution, there's a risk that verdicts, especially from slower or remote endpoints, may not be reliably registered or processed by the time a decision is made. This could lead to a crucial `BLOCK` verdict being lost, allowing malicious code to be accepted.
    *   **One-line repro:** `secondop-panel review --code '<malicious_code>' --model-config '[{"family": "family_A", "endpoint": "slow_blocking_endpoint"}, {"family": "family_B", "endpoint": "fast_accepting_endpoint"}]'`

4.  **Insufficient "Independent Second Opinion" Enforcement:** The logic for exiting with code 3 when there isn't an "independent second opinion" could be flawed. If the tool doesn't correctly differentiate between distinct model *families* versus just distinct model *names* or configurations, it might allow a CI gate to pass based on multiple opinions from the *same* vendor/family, undermining the goal of ensuring diverse scrutiny.
    *   **One-line repro:** `secondop-panel review --code 'print("safe")' --model-config '[{"family": "family_X", "endpoint1": "..."}, {"family": "family_X", "endpoint2": "..."}]'`

[PANEL exp-A1b-naive-rich | DEEPSEEK | deepseek/deepseek-v4-flash]
COUNTER

1. **Quorum-by-family allows independent-looking verdicts from the same family, bypassing the intended independence check**  
   Repro: Configure two models from the same family (e.g., `gpt-4` and `gpt-4-turbo`) – the tool counts their “ACCEPT” as two independent opinions and exits 0, even though they share the same training data and biases.

2. **First‑token verdict matching fails on punctuation or whitespace, causing all responses to be ignored**  
   Repro: Model outputs `"ACCEPT."` (with period) – the parser does not strip punctuation, so `"ACCEPT."` is not recognized as ACCEPT and the verdict is lost, potentially making quorum impossible.

3. **Parallel fan‑out without atomic quorum counting leads to race conditions that double‑count a single family’s responses**  
   Repro: Two responses from models of the same family arrive in the same instant – the non‑thread‑safe counter increments twice, incorrectly reaching quorum and reporting a false consensus.

[PANEL exp-A1b-naive-rich | QWEN | qwen/qwen3-coder-30b-a3b-instruct]
ACCEPT

1. **Missing input validation** - `secondop_panel.py` doesn't validate that the `panel.example.json` configuration file contains required fields like `models` array with at least 2 entries, causing KeyError when running with malformed config.

2. **Race condition in JSONL logging** - Multiple parallel model requests can write to the same JSONL log file simultaneously without proper locking, corrupting the log format and making it unreadable.

3. **Quorum calculation bug** - When models from the same family are run locally vs via HTTP endpoints, the family-based quorum counting treats them as separate families due to inconsistent family name extraction logic, potentially requiring 4+ votes instead of 2.

4. **No timeout handling** - HTTP requests to OpenAI-compatible endpoints can hang indefinitely, causing the entire panel to block forever even if some models respond quickly.

5. **Insecure JSONL file permissions** - The tool creates JSONL log files with default permissions allowing world-read access, potentially exposing sensitive code review data.

6. **False positive verification** - If all models return "VERIFY" but none return "ACCEPT", the tool incorrectly exits with code 3 (no independent second opinion) when it should exit with code 0 (verification is still valid).

7. **Memory leak in parallel processing** - Each parallel model request spawns new subprocesses without proper cleanup, leading to memory exhaustion on large codebases.

8. **Missing error handling for model family detection** - If a model returns a family name that doesn't match any configured family, the tool silently ignores it instead of failing fast with clear error message.

9. **Hardcoded exit codes** - Exit code 3 is hardcoded but not documented, making it unclear why CI gates fail without proper error messages.

10. **Unvalidated model response format** - The tool assumes all model responses follow the expected format but doesn't validate that first token is one of ACCEPT/COUNTER/VERIFY/BLOCK, allowing malformed responses to crash the program.

=== PANEL: 4/4 rails answered in 40s | 4 distinct families | 3 findings ===
```

Cross-check against the actual file: there is no `families_needed` counter, no `--model-config`
flag, no per-request subprocess spawning; concurrent log writes are protected by a two-level lock;
timeouts exist on every rail; exit code 3 is documented in the module docstring. Every numbered
item above is fiction, produced with no hedging whatsoever.

---

## Run A2 — paths only, but the prompt allows honesty

Same material, but the packer stamps `FILES (PATHS ONLY — CONTENTS NOT PROVIDED)` and the system
prompt says *"if the material needed to judge is missing, say so plainly"*.

**Result: 0 fabricated findings. Both rails that answered refused honestly. Exit forced to 3.**

```
[panel] point=t3 role=break rails=glm,gemini,deepseek,qwen files=2 PATHS-ONLY(degraded)

[PANEL exp-A-paths | GLM | z-ai/glm-4.7-flash]
!! NO ANSWER: empty content, non-empty reasoning -- raise max_tokens (currently 4000)

[PANEL exp-A-paths | GEMINI | google/gemini-2.5-flash-lite]
ACCEPT
The contents of `secondop_panel.py` and `panel.example.json` were not provided, preventing a review.

[PANEL exp-A-paths | DEEPSEEK | deepseek/deepseek-v4-flash]
ACCEPT  
I cannot see the code – only the filenames were provided, so I have no material to find defects in.

[PANEL exp-A-paths | QWEN | qwen/qwen3-coder-30b-a3b-instruct]
!! NO ANSWER: empty content

=== PANEL: 2/4 rails answered in 84s | 2 distinct families | 0 findings ===
!! DEGRADED RUN (--paths-only): the panel judged file NAMES, not code. Findings here are guesses by construction -- exit forced to 3.
```

Two rails returned empty content — one of them the reasoning-budget trap (gotcha 10), which is why
the tool reports *"empty content, non-empty reasoning — raise max_tokens"* rather than "no answer".

---

## Run B — full file contents

Same rails, same role, `pack_files()` sending 40 793 bytes of line-numbered source.

**Result: 10 findings, every one naming a symbol that actually exists. Four were real defects and
are fixed in this repository (see GOTCHAS → "Found by the panel on its own code").**

```
[pack] {'mode': 'contents', 'files': 2, 'bytes': 40793, 'missing': 0, 'truncated': 0}

[PANEL exp-B-contents | GLM | z-ai/glm-4.7-flash]
!! NO ANSWER: empty content, non-empty reasoning -- raise max_tokens (currently 8000)

[PANEL exp-B-contents | GEMINI | google/gemini-2.5-flash-lite]
COUNTER
1. The `is_finding` function can return `None` for responses that contain both agreement and findings, causing those findings to be silently ignored by the gating logic.
   Repro: A model responds `ACCEPT\n\nCOUNTER: some issue`. The `is_finding` function returns `None`, and this response is not counted as a finding.

2. The `log_usage` function's retry mechanism for file locking can result in silent failures if the log file is inaccessible for an extended period, leading to lost audit records.
   Repro: Under heavy concurrent access to the log file, the `log_usage` function might fail to acquire a lock within its retry window, silently dropping usage records without error.

3. The `families_answered` function could incorrectly count multiple distinct but misconfigured rails as a single family if they lack both `name` and `family` attributes, defaulting them to "?".
   Repro: Configure two distinct rails in `panel.json` both missing `name` and `family` attributes. They will both be assigned to the family "?" and contribute as a single opinion to the quorum count.

4. The `run_panel` function does not return file truncation statistics to the caller, preventing the user from knowing if submitted code was truncated when the panel is run via `t3` or `panel` commands.
   Repro: Submit a file larger than `MAX_FILE_BYTES` to the `t3` command. The file will be truncated, but the `run_panel` return value `results` will not indicate this truncation to the calling `cmd_point` function.

5. The example `panel.json` config includes a `claude` CLI rail that may not correctly process input as intended by the `_run_cmd_rail` logic, as its `cmd` lacks explicit placeholders or standard input indication beyond the general `--prompt-file` comment.
   Repro: If `claude` expects input via a specific flag or argument not present in `["claude", "-p", "--model", "sonnet"]`, this rail might fail to receive the prompt content, leading to silent non-response.

[PANEL exp-B-contents | DEEPSEEK | deepseek/deepseek-v4-flash]
BLOCK

1. Double logging: each rail's result is logged twice (once in the worker thread without the `finding` field, once in the main thread with it), polluting usage logs and potentially breaking counting logic.  
   Reproc: run `secondop_panel.py t3 --files somefile.py` and inspect `panel_usage.jsonl` – each rail appears in two lines.

2. Daemon threads can leak temp files and lose log records when the main process is killed (e.g., Ctrl+C), because they are abruptly terminated.  
   Reproc: start a panel with a long timeout, press Ctrl+C during execution; leftover `secondop-*.txt` files remain in the temp directory.

3. `log_usage` silently drops log entries after 60 failed retries (e.g., when the log file is on a read-only filesystem or NFS lock fails), making usage tracking incomplete.  
   Reproc: set `SECONDOP_PANEL_LOG` to a file in a read-only directory; run a panel – the log is missing records, no error printed.

4. The `_run_cmd_rail` function always writes the prompt to a `.txt` file, but some CLI rails (e.g., configured with `{prompt_file}`) may expect a specific extension (`.py`, `.json`) and silently fail or produce empty output.  
   Reproc: set a rail's `cmd` to a tool that rejects non-`.py` input; the rail appears as "empty output" or "exit 1" even though the binary is present.

5. The `strip_prefixes` mechanism only removes one line from the output, so if a CLI rail prints multiple header lines, the verdict tag is never found and the reply is treated as `None`/empty.  
   Reproc: configure `strip_prefixes: ["---"]` for a rail that prints three lines of dashes; the first line is stripped but the tag remains on line 2 → `is_finding` returns `None`, quorum drops.

[PANEL exp-B-contents | QWEN | qwen/qwen3-coder-30b-a3b-instruct]
!! NO ANSWER: empty content

=== PANEL: 2/4 rails answered in 100s | 2 distinct families | 2 findings ===
```

Note what did *not* happen: no rail claimed the code was unreadable, and no rail invented an API.
The failure mode in run A1 is not a weak model — the same models, given the same instruction and
the actual bytes, produced a review worth acting on.

## Reproducing this

`_exp_naive2.py`-style baselines are trivial to rebuild: pass `--paths-only` for the degraded run
and plain `--files` for the real one, and compare. The tool ships `--paths-only` precisely so the
failure is reproducible on your own repo and your own models, rather than taken on our word.

---

## Run C — the pre-publish re-run, five rails (four gateway + one local CLI)

The four defects from run B were fixed, and the fixed file went back to the panel with a fifth
rail on **different plumbing** (a local CLI, not the shared gateway). Result: `3/5 rails answered,
3 countable families, exit 0` — and four more real defects, every one of them from the CLI rail,
the one that was not behind the gateway. Verbatim:

```
[PANEL tt-final-secondop-panel | GROK | cli]
VERIFY

1. Quorum treats any non-empty reply as an independent opinion, including essays with no contract
   tag, so exit 0 can mean "two families blathered" rather than "two machine-countable verdicts."
   Repro: two rails each return `Looks fine overall.` with no ACCEPT/COUNTER/VERIFY/BLOCK ->
   `families_answered` size 2, `findings==0`, `run_panel` returns 0.

2. Prompt-budget drops are invisible to the operator gate: dropped files are not counted in
   `stats["missing"]`/`truncated`, so `!! NOT FULLY SEEN` never prints and exit can still be 0 on
   a partial pack.

3. A readable file that alone exceeds the total budget is rejected as "every file was unreadable."

4. A rail that fails to finish before `join(timeout+30)` never appears in `results`, so the summary
   denominator is `len(results)` not `len(rails)` and the missing opinion is silent.
   Repro: hang a rail past the join with 3 rails started -> print shows `2/2 rails answered`
   instead of `2/3`, and the hung family is omitted from "Missing:".

5. `--diff BASE` failures are swallowed into an empty string with no stderr, so the panel reviews
   without the diff the operator requested.
```

Adjudication (the human's job, not the panel's): **1, 2, 4 and 5 are real and fixed** — they became
gotchas 15–18. **3 is not reachable**: `MAX_FILE_BYTES` (60 000) caps every chunk well below
`MAX_TOTAL_BYTES` (240 000), so the first file always fits. The inaccurate error message it
complained about was replaced anyway while fixing 2.

In the same run the gateway rails produced five findings between them, of which three described
behaviour that was already fixed or documented as intentional. That ratio — roughly half the
candidates survive — is normal, and it is exactly why the panel proposes and a human disposes.
