# secondop-panel plugin

One skill, `secondop-panel-review`, that asks a panel of reviewer models from several different labs for a second opinion on a plan, a design fork or a finished change. It runs the published package with `uvx secondop-panel==1.0.1`; the plugin itself contains instructions only.

## What the skill does

- `t1` before coding: is this plan sound?
- `t2` at a design fork: which option, and why?
- `t3` after the build: try to break the change, with full file contents and an optional `git diff`.

Quorum is counted by model family, not by number of reviewers. Exit code `0` means a quorum of distinct families answered; `3` means there was no independent second opinion and must be reported as a warning, never a pass; `2` means bad config or nothing to review.

## Requirements

- `uv` (for `uvx`), or `pipx` as an alternative runner.
- Your own `panel.json`, pointed to by `SECONDOP_PANEL_CONF` or `--conf`. Start from `panel.example.json` in the project repository.
- API keys in environment variables (named by each rail's `key_env`), or logged-in CLIs for `cmd` rails.

## Data

Running a review sends your context note, optional diff and the full contents of the listed files to the providers you configured. Preview it with `--print-prompt`, which calls nobody. The plugin collects nothing. See `PRIVACY.md` and `docs/SECURITY.md` in the repository.

## Install

```
/plugin marketplace add tonydzi/secondop-panel
/plugin install secondop-panel@secondop-panel
```

Source and full documentation: https://github.com/tonydzi/secondop-panel (MIT).
