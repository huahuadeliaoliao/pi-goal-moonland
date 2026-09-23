# Changelog

## Unreleased

- **Prompt-cache friendly redesign** (fixes severe cache misses with goal active — a live session dropped from 88% to 8.5% cache-hit and burned 70M uncached input tokens):
  - The per-call `context` reminder is now a pure function of the objective (status-neutral wording), injected at the **head** of the visible messages whenever a goal record exists — pause/resume/blocked/complete no longer change a single byte. Lifecycle transitions append persistent `goal-status` notes instead. Only create/cancel/replace can invalidate the cache (once each, user-initiated).
  - New periodic **progress ping**: a persistent, tail-appended `goal-progress` message every N goal turns (default 20, `--goal-progress-interval` to tune, 0 disables) carries the live turn count and re-surfaces the objective near the tail. Re-armed after compaction.
  - Continuation prompts are now fully static (no objective, no counters), so stale continuations stay in history unpruned — the `context` hook no longer removes anything mid-history.
  - Tests pin the cache contracts: reminder byte-stability across turns AND lifecycle transitions, prefix-stability of the context output as history grows, ping cadence and compaction re-arm.
- Dropped the `getEntriesByType` feature-detection (pi-lazy-session fork API); state restore uses the stock `getEntries()` scan again.

## 0.1.0 (2026-09-16)

Initial release.

- Goal mode: one persistent objective worked autonomously across turns — a hidden continuation queued on `agent_end`, goal state in session custom entries (compaction-proof), the objective re-injected before every LLM call.
- `/goal` command (status / pause / resume / cancel / replace) and a `goal` tool for the model (get / create / complete / blocked / resume); model-created goals start immediately by default, with `--goal-confirm` as an opt-in approval gate.
- Audit language adapted from kimi-code: evidence-based completion, blocked only after the same blocker repeats for 3 consecutive goal turns.
- Interrupts: Esc pauses; a restored session brings an active goal back paused; the `agent_settled` safety net pauses the goal after exhausted retries.
- Bundled `write-goal` skill; `--goal "<objective>"` flag to start a TUI/RPC session with a goal running.
- Footer status (`goal: active (turn N)`) and inline lifecycle markers in the transcript.
- Packaging: `pi` manifest (extensions + skills) for `pi install`, strict `tsc --noEmit` gate, biome lint/format, node:test suite over a mock `ExtensionAPI`, CI on node 22.19 and 24.
