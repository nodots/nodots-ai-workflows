# Agentic Work Packages: the Cell / Wave / Gate Model

**Status:** documented from practice, 2026-08-10
**Canonical examples:** [nodots/backgammon#360](https://github.com/nodots/backgammon/issues/360) (AI Engine + Plugin Platform — executed, 25+ cells) and [nodots/backgammon#406](https://github.com/nodots/backgammon/issues/406) (Human vs Human Play — planned to the same model). These live in a private repo; §9 is a fictional worked example, invented to the same shape, for readers without access.
**Lineage:** `nodots/auto-shop` (cells, SCOPE.json, phase gates) → `docs/autonomous-claude-system.md` + #15 (daemon-style orchestrator design, superseded) → the live pattern below (Claude coordinator + worktree-isolated subagents)

---

## 1. The model in one paragraph

A large program of work is decomposed into a **parent epic issue** that carries the execution plan, plus one **subissue per cell**. A *cell* is one work package: one bounded feature, one branch, one `SCOPE.json`, one worker agent, one `[READY]` PR. A **coordinator agent** (a Claude session run by the operator) dispatches cells in **waves** — batches of parallel workers whose dependencies are already merged — up to a WIP cap. Phases end in **validation gates** (V0, V1, …): blocking cells that prove the phase's cells integrate before the next phase's dependents dispatch. The coordinator never writes feature code; workers never touch the plan.

## 2. Anatomy of the parent epic

The epic issue is the single source of truth for the run. Both #360 and #406 use the same sections, in order:

1. **Context** — why the program exists; what was established by exploration before planning.
2. **Architecture (summary)** — the boundaries and contracts cells build against.
3. **Agentic execution model** — the dispatch rules (see §4). This section is copied nearly verbatim between epics; it is the reusable "guidelines for dispatching subagents" the coordinator follows.
4. **Definition of done (every cell)** — the uniform checklist inherited by all cells.
5. **Success criteria (epic)** — checkboxes the coordinator ticks as gates pass.
6. **Out of scope** — explicit non-goals with pointers to successor epics.
7. **Workstreams & cells** — phases as bold headings, each cell as a task-list line `- [ ] #NNN · C1.2 Short title`, each phase terminated by a `**VN gate**` line. GitHub renders checkbox progress on the epic automatically.
8. **References** — pattern sources, superseded issues, a designated "cell template" issue (e.g. #406 cites #367).
9. **Decisions log** — dated entries recording irreversible or scoping decisions, with rationale and reversibility notes.
10. **Progress log (live)** — dated entries appended by the coordinator as waves dispatch and merge. In #360 this log records wave contents, per-cell scope-downs made at dispatch time, integration requirements discovered mid-run, and gap cells added.

## 3. Anatomy of a cell (the work package)

Every cell issue follows the template established by #367 (C1.2) and refined in #407 (H0.1):

```markdown
## Cell <ID> — <imperative title>

**Part of** #<epic> · **Phase N · <phase name>** · **Repo:** `<target repo>` (<package path if workspace>)

### Goal
One paragraph: the gap being closed and why. Cites file:line evidence from
the pre-planning exploration, not speculation.

### Scope of work
Bullet list of concrete changes, naming exact files, functions, and
behavioral rules (e.g. "no silent fallback — timeout/5xx throws with
full diagnostics").

### SCOPE.json
{
  "feature": "<kebab-case-name>",
  "project": "nodots-backgammon",
  "branch": "feat/<feature-name>",
  "allowedPaths": ["<globs the worker may edit>"],
  "forbiddenPaths": ["<globs the worker must not touch>"],
  "dependsOn": ["<feature names of prerequisite cells>"],
  "estimatedScope": "<one-line size statement>",
  "execution": {
    "model": "<haiku | sonnet | opus — worker model tier>",
    "pattern": "<solo | implement+review | implement+test | gate-verify>",
    "agents": ["<optional named agent types, e.g. CORE Specialist, e2e-acceptance-tester>"]
  }
}

### Acceptance criteria
- [ ] Behavior-level checks (testable, specific)
- [ ] Tests pass · TypeScript compiles · lint passes
- [ ] `[READY]:` PR opened, linked to this cell

### Dependencies
- **Depends on:** #NNN (or none) · **Parallelizable:** yes — alongside #MMM.
```

Rules embedded in the template:

- **`SCOPE.json` is the isolation contract.** `allowedPaths` bounds the worker's edits; `forbiddenPaths` always includes the frozen contract package, shared config (`package.json`), and CI (`.github/**`). In #360, every cell's `forbiddenPaths` included `packages/backgammon-engine-protocol/**` once the contract froze — parallel agents cannot collide on the interface. Contract changes get their own dedicated cell (label `cell:contract-change`, highest priority), never an ad-hoc edit.
- **Dependencies are declared in three mirrored places:** the `Depends on: #N` line, `SCOPE.json.dependsOn`, and the phase's blocking gate. Redundancy is deliberate — coordinator, worker, and plan reader each consult a different one.
- **Definition of done is uniform:** acceptance criteria met; tests/types/lint green; `SCOPE.json` respected; `[READY]:` PR opened and linked.
- **`execution` is the cell's execution profile** — which model tier the worker runs on and which subagent pattern it uses. The planner assigns a default from the mapping table in §4a; the coordinator resolves the final profile at dispatch (rule 10). The `pattern` vocabulary is closed: `solo` (one worker, no reviewer), `implement+review` (worker spawns a code-reviewer subagent before opening the PR), `implement+test` (Implementer/Tester pair under the same `SCOPE.json`, per rule 9), `gate-verify` (an independent verifier agent proves the integration claim; browser gates use e2e-acceptance-tester). A cell that seems to need a novel pattern is a signal it is cut wrong, the same as a cell that wants to edit its `forbiddenPaths`.

## 4. Dispatch rules (the coordinator's guidelines)

These are the rules stated in the epics' "Agentic execution model" section, plus execution decisions recorded in #360's progress log:

1. **Parallel by default, sequential by declaration.** Independent cells run concurrently up to a **WIP cap of 4**. A cell is queued only when everything in its `dependsOn` is merged.
2. **Waves.** The coordinator dispatches a wave (up to the cap), waits for the wave's PRs to merge, updates the epic checkboxes and progress log, then computes the next wave from the dependency graph. #360's log shows the cadence: "Wave 2 COMPLETE (4/4 merged)… Next wave (deps now met): C1.2, C1.3, C2.3, C4.1."
3. **Isolation via git worktrees.** Each worker runs in its own worktree of the target package repo (`/Users/kenr/Code/nodots-worktrees/<package>/<branch>` per the workspace convention), one `[READY]` PR per cell into that repo's `development`. This is the multi-repo reality: cells target specific package repos, never the workspace as a whole.
4. **Gates are blocking cells.** V0/V1/V2/… are integration proofs (e.g. #365 "contract round-trips end-to-end", #372 "closed engine plays a full game over HTTP"). Downstream phases do not dispatch until the gate passes. Gates that need the human (browser E2E against a deployed stack, production client work) are marked "held for Ken."
5. **Coordinator ⇄ worker via files.** Workers escalate with `BLOCKER.md` and hand off with `HANDOFF.md` in the worktree. The coordinator manages boundaries, unblocks, and sequences merges — it does not write feature code.
6. **Scope-down at dispatch is allowed and logged.** The coordinator may shrink a cell when dispatching it (e.g. #375 "smoke-train on a small sample only; full training deferred to C2.6"; #368 "migration FILE only; no AWS, no deploy"). The scope-down is recorded in the progress log, on the wave entry.
7. **Gap cells are added mid-run.** When a wave exposes missing work (#360: C2.5 revealed the compute kernel was a stub), the coordinator opens a new cell (#393 C2.0k), inserts it into the dependency graph, and logs it. The plan is live, not fixed.
8. **Integration requirements discovered by one cell are pinned to the cells that must honor them.** Example from #360: C1.2 made move validation injectable; the log records "MANDATORY at integration: the wiring cells MUST configure `legalMoveResolver`… Tracked on #372 and #379."
9. **Two levels of fan-out.** Across cells (coordinator drives ≤4 parallel workers) and, where a cell allows, within a cell (a worker spawns Implementer/Tester subagents under the same `SCOPE.json`).
10. **Execution profile is resolved at dispatch.** The cell declares a default in `SCOPE.json.execution`; the coordinator may upgrade or downgrade it when dispatching and records the change in the wave's progress-log entry, the same way scope-downs are recorded (rule 6). Escalation on failure: a cell that files `BLOCKER.md` or fails review twice re-dispatches one model tier up or with a reviewer added — escalation is logged, never silent.

## 4a. Execution profile defaults (the planner's mapping table)

The planner assigns each cell's `execution` block at cell-cut time from this table; the coordinator overrides only at dispatch, per rule 10. This section is copied between epics alongside the "Agentic execution model" section.

| Cell type | Model | Pattern |
|---|---|---|
| `cell:contract-change` | strongest available | `implement+review` — adversarial review before the PR opens |
| Gate cell (VN) | strongest available | `gate-verify` — independent verifier; e2e-acceptance-tester for browser gates |
| Core game-logic feature | session default | `implement+review` |
| Client UI feature | session default | `implement+test` with CLIENT Specialist + e2e-acceptance-tester (the browser E2E requirement in `CLAUDE.md` is mandatory regardless; the profile makes it declarative) |
| Mechanical work (migrations, renames, version bumps, doc moves) | cheapest tier | `solo` |
| Pre-planning exploration | cheap fan-out | read-only Explore agents; not a cell, runs before the epic is written (§7 step 1) |

Deliberately excluded: per-cell token budgets. The WIP cap and wave cadence already bound spend; a budget number in `SCOPE.json` would go stale the way the daemon design did (§6).

## 5. Label lifecycle

Labels on `nodots/backgammon` track each cell's state; the coordinator promotes them:

| Label | Meaning |
|---|---|
| `claude-ready` | Ready for autonomous development (planning done, cell is dispatchable in principle) |
| `cell:queued` | Ready to start, waiting for capacity (deps merged) |
| `cell:active` | Agent currently working |
| `cell:blocked` | Agent stopped, needs coordinator action (`BLOCKER.md` filed) |
| `cell:awaiting-review` | PR open, needs human review |
| `cell:merged` | Complete |
| `cell:contract-change` | Modifies frozen contracts — highest priority, serialized |

## 6. What changed from the original design

`docs/autonomous-claude-system.md` (and issue #15) designed a daemon: an issue-queue manager, spawn scripts, Redis, a monitoring dashboard. That machinery was **not** built. #360's execution decision reads: "running via Claude coordinator + worktree-isolated subagents (one `[READY]` PR per cell), **not** the auto-shop daemon. Lifecycle tracked by `cell:*` labels + these checkboxes." The GitHub issue tracker *is* the queue, dashboard, and audit log; the epic body *is* the orchestrator state. What survived from the daemon design: worktree isolation, the `claude-ready` label, branch-per-issue, quality gates before PR, the WIP cap idea.

## 7. Checklist: standing up a new epic

1. **Explore first.** Establish what already works and what the real gaps are, with file:line evidence (#406's Context section is the model — it lists what is already HvH-ready before listing the four gaps).
2. **Write the epic** with the ten sections of §2. Copy the "Agentic execution model" section from #406 and adjust repo names, WIP cap, and frozen boundaries.
3. **Freeze the contract early.** Whatever interface parallel cells share, land it in Phase 0, tag it, and put it in every later cell's `forbiddenPaths`.
4. **Cut cells** so each is one branch, one repo, one agent, roughly one session of work. Give every cell the §3 template with a complete `SCOPE.json`, including an `execution` profile assigned from the §4a table.
5. **Insert a gate at the end of every phase** whose acceptance is an integration proof, not a unit test. Mark human-required gates explicitly.
6. **Label** all cells `claude-ready`; label the dispatchable ones `cell:queued`.
7. **Dispatch wave 1** (deps-free cells, ≤ WIP cap), one worktree + worker per cell.
8. **On each merge:** tick the epic checkbox, relabel, append to the progress log, recompute the queue, dispatch the next wave.
9. **Log every decision** (scope-downs, gap cells, integration requirements) in the epic body — the epic must remain sufficient to resume the run cold.

## 8. Pointers

- Epic exemplars (private repo): #360 (executed — read its progress log for wave mechanics in practice), #406 (planned — cleanest current template)
- Cell exemplars (private repo): #367 (C1.2, cited as the cell template), #407 (H0.1, latest refinement)
- Public exemplar: §9 below — fictional, complete, same shape as the private originals
- Labels: `gh label list --repo nodots/backgammon` (`cell:*`, `claude-ready`, `epic`)
- Pattern sources: `nodots/auto-shop`; `docs/autonomous-claude-system.md` (historical daemon design)
- Workspace worktree convention + slash commands: `CLAUDE.md` ("Parallel Claude Sessions with Git Worktrees")

## 9. Worked example (fictional)

Everything in this section is invented for illustration. The repo `example/acme-chess`, its issue numbers, and its cells do not exist; they mirror the structure of the private exemplars one-to-one. The program: adding online multiplayer to an existing single-player chess app.

### 9.1 The parent epic — `example/acme-chess#100`

Excerpted; the ten §2 sections appear in order. The workstreams section:

```markdown
## Workstreams & cells

**Phase 0 · Contract**
- [x] #101 · M0.1 Freeze the move-protocol package (types + wire format)
- [x] #102 · V0 gate — protocol round-trips: encode → send → decode → identical position

**Phase 1 · Server**
- [x] #103 · M1.1 Game session store (create/join/expire)
- [x] #104 · M1.2 WebSocket relay with reconnect
- [ ] #105 · M1.3 Server-side move validation against the frozen protocol
- [ ] #106 · V1 gate — two headless clients complete a full game over WS

**Phase 2 · Client**
- [ ] #107 · M2.1 Lobby UI (create/join)
- [ ] #108 · M2.2 Live board sync + opponent clock
- [ ] #109 · V2 gate — browser E2E: two real browsers, one game, held for human
```

Decisions log and progress log entries, showing the rule 6 / rule 7 / rule 10 mechanics:

```markdown
## Decisions log
- 2026-08-03 — Protocol frozen at v0.1 after #101; all later cells carry
  `packages/move-protocol/**` in forbiddenPaths. Reversible only via a
  cell:contract-change cell.

## Progress log (live)
- 2026-08-04 — Wave 1 dispatched: #103, #104 (deps-free, cap 4, 2 used).
  Scope-down on #104 at dispatch: reconnect only; presence indicators
  deferred to a Phase 2 gap cell if needed.
- 2026-08-05 — Wave 1 COMPLETE (2/2 merged). #104 revealed the session
  store has no expiry sweep — gap cell opened: #110 M1.1k, inserted
  before V1. Next wave: #105, #110.
- 2026-08-06 — #105 failed review twice (validation accepted illegal
  castling). Re-dispatched per rule 10: model upgraded one tier,
  pattern upgraded solo → implement+review. Logged here; SCOPE.json
  default unchanged.
```

### 9.2 A cell — `example/acme-chess#105`

```markdown
## Cell M1.3 — Validate moves server-side against the frozen protocol

**Part of** #100 · **Phase 1 · Server** · **Repo:** `example/acme-chess-server`

### Goal
The relay (#104) forwards any syntactically valid move; an illegal move
desyncs both clients. `relay.ts:88` forwards without consulting the
rules engine. Close that gap: every inbound move is validated before
broadcast, and an illegal move is rejected with a typed error — no
silent drop.

### Scope of work
- Add `validateMove(position, move)` call in `relay.ts` before broadcast
- Reject with `MoveRejected { reason, position }` over the wire — never
  disconnect, never silently drop
- Property test: generated random games, every broadcast move is legal

### SCOPE.json
{
  "feature": "server-move-validation",
  "project": "acme-chess",
  "branch": "feat/server-move-validation",
  "allowedPaths": ["src/relay/**", "src/validation/**", "test/**"],
  "forbiddenPaths": ["packages/move-protocol/**", "package.json", ".github/**"],
  "dependsOn": ["websocket-relay"],
  "estimatedScope": "one session; ~3 files plus tests",
  "execution": {
    "model": "sonnet",
    "pattern": "solo",
    "agents": []
  }
}

### Acceptance criteria
- [ ] Illegal move from a client produces MoveRejected, game state unchanged
- [ ] Property test passes: no illegal move is ever broadcast
- [ ] Tests pass · TypeScript compiles · lint passes
- [ ] `[READY]:` PR opened, linked to this cell

### Dependencies
- **Depends on:** #104 · **Parallelizable:** yes — alongside #110.
```

Note the arc across the two excerpts: #105 was cut as `solo` on the default tier (the §4a mapping for a core-logic cell would have said `implement+review`; the planner under-provisioned it), it failed review twice, and rule 10 escalated it at re-dispatch — with the escalation recorded in the progress log, not by editing the cell.
