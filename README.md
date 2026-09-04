# nodots-ai-workflows

Patterns for running programs of work with coordinated AI agents, documented from practice at [Nodots](https://nodots.com).

## Contents

- [Agentic Work Packages: the Cell / Wave / Gate Model](agentic-work-packages.md) — decompose a large program into a parent epic plus one issue per *cell* (one bounded feature, one branch, one `SCOPE.json`, one worker agent, one PR), dispatched in dependency-ordered *waves* by a coordinator agent, with blocking validation *gates* between phases. Includes the per-cell execution profile (model tier + subagent pattern), a protocol for drafting the plan itself with AI (§7a), and a complete fictional worked example.
- [Working Agreements for Agent-Executed Work](working-agreements.md) — how a worker is expected to *execute and prove* a work package: test-first as the default, dead gates (skipped tests, disabled CI, artifact-only deploy checks), the ban on silent fallbacks, boundary round-trip tests, multi-repo release sequencing, hook-enforced scope and handoff, destructive-command rules, and reporting discipline. Each rule names the failure that produced it.

## Status

Documented from practice, 2026-08-10 (work packages) and 2026-09-04 (working agreements). The canonical epics this pattern was extracted from live in a private repository; the worked example in §9 of the doc is fictional and self-contained.

## License

[MIT](LICENSE). Copy the templates into your own repos, public or private, without attribution friction.
