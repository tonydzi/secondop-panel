# Privacy

The Claude Code plugin in `plugin/` is instructions only. It contains no code, collects no data and sends nothing anywhere by itself.

When you follow its instructions and run `secondop-panel`, the tool sends the prompt it builds (your `--context` note, an optional `git diff`, and the full contents of the files you pass with `--files`) to the reviewer endpoints and CLIs that you configured in your own `panel.json`. Those providers handle that data under their own terms. Nothing goes to the author of this project.

API keys are read from environment variables only and are never written to files or logs. The local usage log (`panel_usage.jsonl`, or `$SECONDOP_PANEL_LOG`) records rail name, model family, model, duration and verdict, never the prompt and never a credential.

See `docs/SECURITY.md` for what leaves your machine and how to preview it with `--print-prompt`.
