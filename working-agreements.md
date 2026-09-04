# Working Agreements for Agent-Executed Work

**Status:** documented from practice, 2026-09-04
**Companion to:** [agentic-work-packages.md](agentic-work-packages.md) — that document says how work is *decomposed and dispatched*; this one says how a worker is expected to *execute and prove* it.
**Source:** rules that accumulated in the Nodots repos as `CLAUDE.md` entries, agent definitions, `.claude/` hooks, and post-incident notes. Each section names the failure that produced the rule. None of them is hypothetical.

---

## 1. Why this document exists

The cell/wave/gate model bounds *what* an agent may touch. It says nothing about what counts as finished. In practice most of the damage came from cells that were dispatched correctly, stayed inside `allowedPaths`, opened a green PR — and shipped something broken, because "tests pass" was satisfied by tests that could not fail.

Every rule below is a rule about **evidence**. The cell template's definition of done (`acceptance criteria met; tests/types/lint green; SCOPE.json respected; PR opened`) is necessary and not sufficient. This document is the sufficient part.

---

## 2. Test-first is the default execution mode

A worker writes the failing test before the fix.

- **The test must fail before the change and pass after.** A test written after a fix, against the fixed code, proves the code runs. It does not prove the bug is gone. If a worker cannot make a test fail against the pre-change tree, it does not yet understand the bug — that is a `BLOCKER.md`, not a licence to proceed.
- **Every bug fix ships with a regression fixture built from the real failing input.** Production position, production payload, production row — not a hand-written approximation of one. Approximations of the failing input are how a bug class gets "fixed" three times.
- **UI cells are not done without a browser-driven end-to-end test that exercises the acceptance criteria.** Not a unit test with a mocked DOM. The rule in the source repos is absolute: no client task is complete without an E2E test that shows the criteria met, and nothing is claimed fixed until that test has been run and seen to pass.
- **"Should now work" is not a status.** A worker reports what it ran and what the run printed. If it did not run, it says so.

This is the worker-side counterpart to the gate rule in the dispatch model: gates prove integration, tests prove the cell. Both are proofs, not assertions.

### 2a. Test-writing rules that came from specific failures

- **A compile error in a test is a test failure.** Type errors in test files get fixed, never suppressed or excluded from the run. They hide the same defects the tests were written to catch.
- **Tests must be deterministic and order-independent.** A cell's tests run in a fleet, in parallel, on machines that are not the author's.
- **Test the state machine from both sides.** Where a domain has symmetric actors (two players, two directions, two tenants), a test that only exercises one side will pass while the other is inverted. A whole class of production bugs in the source project was one actor's coordinates used for the other's.

---

## 3. A gate that has never run is not a gate

The most expensive incidents in the source project were not missing tests. They were tests that existed, were correct, and never executed.

- **Never gate a test behind an environment flag nothing sets.** A regression test for a stuck-robot bug sat behind `RUN_GNUBG_HINTS=1` for a year. Nothing in CI, and no developer, ever set it. The bug recurred in production with its own regression test sitting green-and-skipped in the repo. If a test needs a resource CI lacks, the correct outcome is to give CI the resource or to fail the run loudly — never to skip silently.
- **Verify CI is enabled before trusting it.** One repo's workflows had been manually disabled for three months; every merge in that window shipped unchecked. Two other repos had no workflows at all, so their state was *unverifiable*, not *good*. Before a wave dispatches into a repo, confirm that repo's CI actually runs, and that its last run was for a real commit.
- **Count skips.** A suite that reports `40 passed, 18 skipped` is reporting a problem. Coverage that differs between two machines means neither machine runs everything — establish which suites run where before quoting a coverage number.
- **A red pipeline that nobody is alerted about is a dead gate too.** Three of these ran simultaneously in one project: disabled CI, a skipped regression suite, and a deploy job that had failed on every attempt for eleven days. Alerting on gate failure belongs in the plan, as a cell.

---

## 4. Fail loudly; silent fallbacks are prohibited

**Never add fallback logic that substitutes a different answer when the intended one cannot be produced.** Throw, log the full diagnostic context (inputs, the candidate that failed, the alternatives considered), and stop.

The reasoning is specific to agent-built systems. A fallback converts a *detectable* failure into a *plausible-looking wrong answer*. In the source project, a fallback in move execution let a robot play legal-but-random moves for months while every health check stayed green; the bug was only found by measuring play strength. When the fallback was removed, the same defect surfaced within a day as a loud, diagnosable stall.

Corollaries:

- A retry loop is a recovery mechanism only if the underlying operation can succeed. Around a deterministic failure it is an infinite loop that looks like activity.
- Prefer the failure that stops the system over the failure that degrades it, in any system where correctness is checkable but not checked continuously.
- This rule outranks user-facing smoothness. It is stated in the source repo as an absolute, in capitals, because it was violated by well-meaning defensive code more than once.

---

## 5. Test at the boundaries where things disappear silently

Three boundaries in the source project each swallowed a field or a version without any error. All three now have a mandatory check.

- **Persistence.** A new field on a domain object requires a save → read → update round-trip test *before merge*. A field added to a type that is stored as explicit columns is silently dropped on every write; the design assumption "an optional field flows through JSON persistence untouched" was asserted and never verified, and shipped a broken user-facing flow to production. If a repo's storage layer maps rows to objects in more than one place, the test must prove every mapper.
- **Serialization.** Cross-process boundaries mutate values in ways types do not describe (`undefined` becoming `null` through JSON, a `Set` arriving as an array). Assert on what comes out of the wire, not on what went in.
- **Declared dependency ranges.** Code that needs a new symbol from a sibling package must have the *declared range* bumped, not just the lockfile refreshed. A range that still satisfies the old locked version means the package manager never moves, CI installs from the lock, and the build dies in the type checker while every developer machine passes on a hand-refreshed tree. `--package-lock-only` cannot fix this; the range itself must change.

Generalization for planners: **any cell that adds a field, a type, or a dependency crosses at least one of these boundaries, and its acceptance criteria must name the round-trip test explicitly.** "Tests pass" will not catch a boundary that has no test.

---

## 6. Deploy verification means exercising the feature

Artifacts are not evidence. Image digests, health endpoints, bundle hashes and a green pipeline collectively prove that *something* deployed.

- **A user-facing flow is unverified until the flow has been run once, end to end, after the deploy.** In the incident that produced this rule, post-deploy verification checked every artifact and nobody executed the feature — the user found it broken.
- **Where no staging environment exists, say so and ask.** The honest options are a single sanctioned production exercise or an explicit accepted risk stated in one line. Both are the operator's call, not the agent's.
- **Never propose changing production configuration to make a local test possible.** (Production auth callbacks, DNS, cloud account settings.) Find another route or state the residual risk.
- **Verify the account/target before any command that touches infrastructure.** Identity check first, compare against the expected value, and stop if it does not match. Deploying to the wrong account is not recoverable by a revert.

---

## 7. Multi-repo mechanics

Where cells target several independent repos (the normal case in the source project — see the dispatch model's rule 3), sequencing is part of the plan, not an implementation detail.

- **Merge in dependency order.** Types → core → consumers → services → clients. A cell whose dependency is merged but *not released* is still blocked.
- **A cross-repo change is not visible to consumers until it is published.** Repos consume each other as published packages; a commit is not a delivery. Cells that span repos need one PR per repo plus an explicit release step, and the release step is its own cell.
- **Promote to the release branch before publishing.** Publishing from the working branch leaves the registry pointing at a commit that is not on the release branch, entangles version numbers, and trips branch-drift checks. Promotion must be a fast-forward; a merge commit puts the branches one apart and fails the same check.
- **The trigger repo must be pushed last.** Where a deploy workflow re-clones each dependency at its current release branch, pushing the trigger first builds the *old* dependency code. Push every dependency, verify, then push the trigger.
- **Never chase history across a repo boundary.** Use the history of the repo the file lives in. Never restructure a multi-repo workspace into a single repo to make a tool happy.
- **Confirm which repo the shell is in before running anything.** In a workspace of sibling repos, the previous cell's directory is the most common source of a command run in the wrong place.

---

## 8. Enforcement beats instruction

Rules written in a prompt decay; rules enforced by a hook do not. The source repos enforce three things mechanically, and the mechanism is worth copying before the prose is.

- **`SCOPE.json` is enforced by a `PreToolUse` hook** that blocks any edit or write outside `allowedPaths` or matching `forbiddenPaths`. A worker physically cannot leave its lane. The hook fails open and only intercepts file writes — it is a coordination guardrail, not a security control. Coordination files (`SCOPE.json`, `HANDOFF.md`, `BLOCKER.md`) are always writable, and with no `SCOPE.json` present there is no restriction, so ordinary work on the mainline is unaffected.
- **Handoff is enforced by a `Stop` hook.** On a feature branch, with uncommitted changes, and no `HANDOFF.md` or `BLOCKER.md` present, the session is blocked from ending. This is what makes "the coordinator is not available on demand" survivable: a worker that walks away leaves a written state, or does not walk away.
- **The worker's contract is a checked-in agent definition, not a dispatch-time prompt.** It states, in this order: read `SCOPE.json` first; read `HANDOFF.md` if resuming; explore before writing; the stopping conditions; the completion protocol. Because it lives in the repo, every worker in that repo gets the same contract and it is reviewed like code.

### 8a. Stopping conditions (worker side)

A worker stops and writes `BLOCKER.md` when:

- it needs to edit a path outside `allowedPaths`, or a frozen config file;
- **tests still fail after two distinct fix attempts** — and it describes both attempts;
- it faces an architectural decision with no clear answer in the cell;
- `SCOPE.json` looks wrong or incomplete.

The response to a scope violation is to stop, never to widen the lane. A cell that wants to edit its own `forbiddenPaths` is a cell that was cut wrong.

### 8b. Durable state over conversational state

Sessions compact. Decisions, findings and measurements go into files — `HANDOFF.md`, the epic's progress log, test names, code comments — not into the conversation alone. The test for a completion protocol is whether a cold session could resume from what was written.

---

## 9. Destructive commands

- **Never pattern-kill a process fleet by substring.** `pkill -f '<script> generate 2'` matches `generate 200`; it killed a twenty-worker run fifty minutes in, and only the shards that had flushed survived. Kill by PID captured at launch (workers log their PID to a status file), or anchor the pattern so it cannot match a longer argument. **Always `pgrep -af` and read the matched lines before any kill.** A match count larger than expected is evidence that the pattern is too broad, not that there is more to clean up.
- **`git stash` is banned in worker worktrees** — see dispatch rule 3 in the companion document for the mechanism. Capture a patch outside the tree instead.
- **Look at the target before deleting or overwriting it**, and prefer branching from the remote over juggling local state.

---

## 10. Verify claims against primary sources before building on them

The cell template's "cites file:line evidence, not speculation" rule extends past the codebase.

Before any analysis or plan that rests on an external fact — who owns a domain, a company, a trademark, a package name; what a registry, a spec, or an API actually says — run the primary-source lookup **first**. A fact marked "unverified" in a planning document is not licensed for use just because it is labelled; if the check is cheap, **the check is the next action, never the inference**.

The failure that produced this rule: an entire competitor analysis was built on an ownership claim that the public registry record contradicted, one command away, for the whole time. The cost was not embarrassment — it was a favour spent with a real contact to establish what a free lookup already showed.

---

## 11. Reporting to the operator

The operator is reading many parallel workers. Report length is a cost they pay.

- **Bullets by default**, for findings, status and options alike. Prose is the exception.
- **A yes/no question gets a yes/no answer**, first, before any reasoning.
- **Lead with the ask.** When the operator must act, the steps go at the top as a short checklist, not buried under context.
- **Depth of investigation does not license length of report.** Two worktrees and four probes still produce roughly twenty bullets.
- **No framing or transition sentences.** "Two things worth naming", "here's what's actually in the way" — every one is a line the reader discards to reach the content. Bullets need no introduction.
- **One question at a time.** Batching decisions into a table is the same dump in another shape.
- **Be unambiguous about bug state.** Distinguish *discovered*, *under investigation*, *fixed*, and *verified fixed* explicitly and never let the language blur them. "Fixed" without a run behind it is the single most expensive word an agent can use.

---

## 12. Domain invariants belong in the repo, not in the prompt

Every project has a handful of rules that are load-bearing, non-obvious, and violated identically by every new contributor — human or agent. In the source project these include a canonical way to resolve a position for the active actor, a single authoritative source for game state, and a strict rule that all domain logic lives in one package while the service layer only validates, persists and relays.

Three things make them stick:

1. **Write them where the tools load them** — in the repo's own `CLAUDE.md`/`AGENTS.md`, at the top, stated as absolutes with the reason attached. A rule in a sibling repo's file does not load for the repo being edited; this was discovered the hard way in a workspace of siblings.
2. **Name the wrong version alongside the right one.** "Use X, not Y" is followed; "use X" is followed until Y looks convenient.
3. **Give each one a test that fails when it is violated**, so the invariant is enforced at the same place as everything else in this document.

Style rules earn their place the same way: they are worth writing down only when a specific defect traces back to their absence — a permissive cast that hid a state bug, a duplicated local type that drifted from the shared package, a comment that described *what* instead of *why*. An explicit cast is a code smell that requires a comment justifying it; a duplicated type is a state bug waiting for a version bump.

---

## 13. Checklist: adding these to an epic

- Cell acceptance criteria name the **failing test first**, and for UI cells, the **browser E2E** that proves the criteria (§2).
- Any cell that adds a field, type or dependency names its **boundary round-trip test** (§5).
- The repo the wave targets has **CI enabled, running, and free of silently skipped suites** — verify before the wave, not after (§3).
- `SCOPE.json` enforcement and the handoff hook are **installed in the target repo** before workers dispatch (§8).
- Release and deploy **sequencing is a cell**, in dependency order, with the trigger repo last (§7).
- Deploy gates require the **feature to be exercised**, not the artifacts inspected (§6).
- The epic states the project's **domain invariants** or links the repo file that does (§12).
