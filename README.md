<p align="center">
  <img src="https://raw.githubusercontent.com/huahuadeliaoliao/pi-goal-moonland/main/docs/logo.png" width="640" alt="pi-goal-moonland — moon phases ending in a target">
</p>

<h1 align="center">pi-goal-moonland</h1>

<p align="center">
  Goal mode for the <a href="https://github.com/earendil-works/pi">pi</a> coding agent:<br>
  one persistent objective, worked autonomously across turns until it is verifiably done.<br>
  <a href="https://github.com/MoonshotAI/kimi-code">kimi-code</a>-style goals, rebuilt as a pi extension.
</p>

<p align="center">
  🌑 🌒 🌓 🌔 🌕<br>
  <sub>one moon phase per goal turn — a nod to kimi-code's TUI moon loader</sub>
</p>

## What is a goal?

A normal prompt tells the agent what to do *next*. A goal says what must become *true*:

```
/goal Fix every failing test in test/auth, rerun npm test until it exits 0,
      and stop and report if a failure needs changes outside test/ or src/auth
```

Once a goal is active, the agent keeps working. Each time a run ends with the goal still
active, pi-goal-moonland queues a hidden continuation and the agent picks the work back up — slice
by slice, across as many runs as it takes. The loop ends only when the agent marks the goal
`complete` (with evidence) or `blocked` (with a concrete reason), or when you stop it.

## Install

Requires **pi ≥ 0.87** — the continuation loop runs on pi's actionable
`agent_before_settle` boundary; older hosts get an error notification instead.

```sh
pi install git:github.com/huahuadeliaoliao/pi-goal-moonland  # from git
pi install npm:pi-goal-moonland                             # from npm
```

The project was previously called `pi-goal`; the npm package name is unchanged.
Existing git installs can keep the old source URL through GitHub's redirect. Do not add
the new git URL alongside the old one: Pi treats them as different package sources.
The `/goal` commands, `goal` tool and saved goal state are unchanged.

Or ad-hoc, without installing: `pi -e /path/to/pi-goal-moonland` (the directory or `index.ts` both work).

Manual copy still works too:

```sh
git clone https://github.com/huahuadeliaoliao/pi-goal-moonland.git
cp -r pi-goal-moonland ~/.pi/agent/extensions/goal
```

## Usage

| Command | Action |
| --- | --- |
| `/goal <objective>` | Start a goal (bare `/goal` with no goal opens an editor) |
| `/goal status` | Show the current goal |
| `/goal pause` | Pause after the current run finishes |
| `/goal resume` | Resume a paused/blocked goal (starts a run when idle) |
| `/goal cancel` | Drop the goal and abort goal-driven work |
| `/goal replace <objective>` | Drop the current goal and start a new one |

The agent can also start goals itself via its `goal` tool. Say *"help me write a goal"* and the
bundled **write-goal** skill walks the agent through drafting a verifiable completion contract —
you approve the wording in conversation, and the goal starts without a second prompt.

Flags:

- `--goal "<objective>"` — start a TUI/RPC session with a goal already running
- `--goal-confirm` — ask for your approval before an agent-created goal starts
- `--goal-progress-interval <N>` — append a persistent goal-progress message every N
  goal turns (default 20; 0 disables)

Headless: `pi -p "/goal <objective>"` runs the goal to completion, then exits.

**Security note:** a goal is an unattended loop that edits files and runs commands, and
agent-created goals start immediately by default. If you run pi-goal-moonland against untrusted content
or simply want a human checkpoint, start pi with `--goal-confirm`.

## How it works

- **Continuation loop** — at pi's `agent_before_settle` boundary (after retries,
  auto-compaction and your queued messages are done), if the goal is still active, a
  hidden continuation carrying the objective, the live turn count and the audit rules
  is appended and pi makes one more model call. No timers. An exhausted error pauses
  the goal.
- **Prompt-cache friendly** — every goal message the model sees is persistent and
  appended at the end of the conversation: the goal reminder at start, status notes
  (paused/cancelled), continuations, and periodic `goal-progress` pings. Nothing is
  injected per call, edited or removed, so creating, pausing, resuming, completing,
  cancelling or replacing a goal only extends the cached prefix — including codex
  response chaining. Mid-run notes are queued for the turn boundary, never steered in.
- **Compaction-proof** — goal state lives in session custom entries (never enters LLM
  context). After a compaction the goal reminder is appended again (free: compaction
  already resets the cache), and every continuation repeats the objective anyway.
- **Branch-aware** — goal state is read from the current session branch, so `/tree`
  navigation and forks see the goal as it was at that point.
- **Honest endings** — the prompt wording (adapted from kimi-code's iterated audit
  language) pushes the agent to verify against current evidence before `complete`, and
  to only declare `blocked` after the same blocker repeats for 3 consecutive goal turns.
- **Interrupts** — Esc pauses the goal; resuming a session or switching branches
  restores an active goal as paused; explicit `/goal resume` continues it.
- **Visibility** — a footer status (`goal: active (turn N)`), inline lifecycle markers
  (started/paused/complete/blocked), hidden continuation messages.

## Development

```bash
npm install
npm test          # node:test with a mock ExtensionAPI — no real pi process
npm run check     # tsc --noEmit
npm run lint      # biome check
```

The extension is plain TypeScript loaded by pi via jiti — no build step. `@earendil-works/pi-*` and
`typebox` are peer dependencies (pi bundles them for extensions); they are pinned in `devDependencies`
only for tests and type-checking.

## Design provenance

The design is ported from [kimi-code](https://github.com/MoonshotAI/kimi-code)'s goal
feature (`packages/agent-core-v2/src/features/goal`, MIT), with
[oh-my-pi](https://github.com/can1357/oh-my-pi) (MIT) as a cross-reference. Deliberately
dropped: token/turn/wall-clock budgets and the `/goal next` queue — the goal is the goal.
Re-add them if real usage asks.

## License

[MIT](LICENSE) © 2026 huahuadeliaoliao
