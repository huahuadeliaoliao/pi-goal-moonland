# AGENTS.md

Guidance for agents working in this repository.

## What this is

`pi-goal-moonland` — goal mode for the pi coding agent, shipped as a pi extension (`index.ts`, no build step; pi loads TypeScript via jiti). A goal says what must become *true*; after each agent run ends with the goal still active, the extension queues a hidden continuation so the agent keeps working across runs until the model marks the goal complete/blocked via the `goal` tool.

Layout: `index.ts` (extension), `state.ts` (pure state module), `prompts.ts` (prompt templates), `skills/write-goal/` (bundled skill).

## Commands

- `npm test` — node:test suites in `test/`; they mock `ExtensionAPI`/`ExtensionContext` and never touch a real pi process.
- `npm run check` — `tsc --noEmit` (strict). jiti has no build step: this is the only compile check.
- `npm run lint` — `biome check .` (must be clean, warnings included).

## Invariants

These break silently if changed; the test suite pins most of them.

- **Goal state lives in session custom entries** (`goal-state`), outside LLM context, read from the **current branch** (`sessionManager.getBranch()`, re-read on `session_tree`). An active goal always comes back paused on restore or branch switch.
- **Cache contract (load-bearing, 2026-09): append-only, no per-call injection.** There is no `context`/`context_with_system` hook, and boundary drafts are only `custom_message`s — never `context_edit` or compaction. Every model-visible goal message is persistent and appended at the tail: `goal-reminder` (user-started goal; re-append after compaction), `goal-status` (paused/cancelled), `goal-continuation`, `goal-progress`. History: a tail injection with a per-turn counter broke codex `previous_response_id` chaining (hit 88% → 8.5%); a status-neutral head injection still cost a full-context miss on every create/cancel/replace (measured: cacheWrite 371k on one create). Appends are prefix-extensions on every provider. Live data (turn counts, status) is fine in appended messages because they are written once.
- **Never steer.** Mid-run `pi.sendMessage` without options becomes a steer (extra model call on the final turn; a steer queued during an aborted run leaks into the next run). All goal notes go through `appendNote` (`triggerTurn: false`: appended when idle, flushed at the turn boundary mid-run) or boundary drafts. The idle create reminder uses `deliverAs: "nextTurn"` so it rides the objective's prompt; `/goal resume` from idle kicks with `triggerTurn: true`.
- **The continuation loop is the product.** `agent_before_settle` with an active goal and outcome `completed` returns `{ entries: [...event.entries, continuationDraft], continue: true }`. Outcome `error` (retries already exhausted) pauses; `aborted` pauses. `agent_end` only pauses on an aborted stop and persists the counter. `agent_settled` stays as a safety net. Requires pi >= 0.87 (`hostSupportsBoundaries(VERSION)` warns on older hosts).
- **Compaction.** `session_compact` outside a run (idle `/compact`) appends the status-aware reminder at once; inside a run it arms `reminderArmed`, consumed by the next `turn_end` (draft) or `agent_before_settle` (draft for paused/blocked goals; an active goal's continuation already carries the objective). Completed goals get no reminder. `inRun` spans `agent_start` → `agent_settled`.
- **Progress pings.** Every N goal turns (default 20, `--goal-progress-interval`, 0 disables) `turn_end` returns a `goal-progress` draft (no `continue`, so no extra model call). They matter for long runs with many tool turns between continuations. Pings never affect turn counting.
- **Honest endings.** The audit language in `prompts.ts` is adapted from kimi-code's iterated prompt wording — edit conservatively.
- **Model-created goals start without a confirmation prompt** (explicit maintainer decision, 2026-09, pre-0.1.0): the write-goal flow's checkpoint is conversational — show the draft, get approval, then create. `--goal-confirm` re-enables the TUI gate for users who want one. Do not restore the gate by default.
- **Turn counting is in-memory between runs.** State persists on status changes, at run boundaries (`agent_end`), and on `session_shutdown` — not on every `turn_end`. The counter is not cosmetic: the continuation's and progress ping's turn count and the 3-turn blocked audit both anchor on it (the model can also read it live via the goal tool's `get`). A hard kill (SIGKILL) under-reports it and a blocked declaration may land slightly late after a crash; accepted, because quit/reload/SIGTERM/SIGHUP all emit `session_shutdown` (verified in pi's runtime) and per-turn writes to survive SIGKILL are a poor trade.
- **Interrupted or errored runs never continue the loop.** Abort pauses (at `agent_end`); an error pauses at `agent_before_settle`, which pi only reaches after its own retries.
- **The objective is untrusted text** and goes through `escapeUntrusted` before entering any prompt.
- `state.ts` stays pure (no pi APIs) so it remains unit-testable.

## Packaging

- `@earendil-works/pi-*` and `typebox` stay in `peerDependencies` with `"*"` and `optional: true`; pinned versions live in `devDependencies` for tests and type-checking only. pi resolves these imports to its own bundled modules at load time.
- The package is multi-file: the `files` whitelist in package.json must keep covering `index.ts`, `state.ts`, `prompts.ts` and `skills/`, or installs silently lose functionality. CI runs `npm pack --dry-run` to catch regressions.

## Testing notes

- `test/helpers.ts` builds the mock harness: `createMockPi()` captures registrations (handlers, tools, commands, flags, renderers) and sent messages; `createMockCtx()` fakes ui/idle/session entries; `fireAsync()` drives event handlers.
- `MockCtxOptions.confirm` answers the goal-creation confirm dialog (only reached with `--goal-confirm` on); `editorText` answers the bare `/goal` editor; `idle: false` exercises the steer/abort paths.
