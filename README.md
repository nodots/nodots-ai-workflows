# nodots-ai-workflows

Patterns for running programs of work with coordinated AI agents, documented from practice at [Nodots](https://nodots.com).

## Contents

- [Agentic Work Packages: the Cell / Wave / Gate Model](agentic-work-packages.md) — decompose a large program into a parent epic plus one issue per *cell* (one bounded feature, one branch, one `SCOPE.json`, one worker agent, one PR), dispatched in dependency-ordered *waves* by a coordinator agent, with blocking validation *gates* between phases. Includes the per-cell execution profile (model tier + subagent pattern) and a complete fictional worked example.

## Status

Documented from practice, 2026-08-10. The canonical epics this pattern was extracted from live in a private repository; the worked example in §9 of the doc is fictional and self-contained.
