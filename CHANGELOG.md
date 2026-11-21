# Changelog

What shipped, in plain words. This file did not exist before 2026-11-06: the entry for 0.2.0 is
recorded after the fact from the tag, not backdated.

## [Unreleased]

## [1.0.2] - 2026-11-21

### Changed
- **Reviewer prompt gets an evidence rule.** A rail that was given only a path or a summary used to
  "review" files it had never seen and invent names. The system prompt now tells it to say
  "no artifact body given" and stop, and to never cite files or functions absent from the text.

## [1.0.1] - 2026-11-06

### Fixed
- **A diff and the files now share one prompt budget.** Before, a large `--diff` plus `--files`
  could push the prompt over the limit. Now the cut is announced ("NOT FULLY SEEN") and a diff
  that uses up the budget no longer makes the tool claim every file was unreadable.
- **Package version matches the CLI.** `pyproject.toml` said 0.x while the CLI already reported
  1.0; both now say 1.0.1, and the tag is `v1.0.1`.

## [0.2.0] - 2026-10-02

Recorded after the fact. See the `v0.2.0` release notes.
