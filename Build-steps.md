# Hooks in depth

This file describes the **steps followed** to add Concept 20 (**Hooks in depth**) to the Claude Agent SDK Lab:
what was read, what was decided, how it was tested, and what the tests changed.
To learn the concept itself, read [Tab20-Hooks-in-depth.md](Tab20-Hooks-in-depth.md).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 20 | `HOOK_EVENTS`, `UserPromptSubmit`, `PermissionRequest`, `updatedToolOutput`, `PostToolUseFailure`, `Stop`, `SubagentStart` / `SubagentStop`, `systemMessage`, `continue: false`, `timeout`, `{ async: true }` | `/api/c20/run`, `/files`, `/file`, `/reset` | [Tab20-Hooks-in-depth.md](Tab20-Hooks-in-depth.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample19, so `npm install` is not needed. Open the **20. Hooks in depth** tab.
Start it from a normal terminal: from inside Claude Code, the inherited `CLAUDE_CODE_*` variables make every run use
the login (`apiKeySource: "none"`, see Tab16).

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

Again, the request said "implement the following feature sample" with no feature text. Four topics were offered:
**Slash commands**, **Hooks in depth**, **Plugins in depth**, or a pasted spec. **Hooks in depth** was chosen:
Concept 7 only covered `PreToolUse` / `PostToolUse`.

## Step 2: Read the existing samples

| Read | To learn |
|---|---|
| `server/concepts/07-hooks.ts`, `Concept07Hooks.tsx`, `Tab7-Hooks.md` | What Concept 7 covers, the `traced()` wrapper, the hook table UI |
| `server/concepts/19-session-management.ts`, `Concept19SessionManagement.tsx` | The latest `BASE` options, a lab folder, input checks, the doc style |
| `server/concepts/08-subagents.ts` | `agents` and the rule that a subagent's tools must be in the session's pool |
| `server/index.ts`, `server/sse.ts`, `src/lib/sse.ts`, `src/App.tsx`, `src/styles.css`, `.gitignore` | Mounting, SSE, the tab list, CSS to reuse |

## Step 3: Check the types

`sdk.d.ts` (`0.3.281`): `HOOK_EVENTS` (33 names), `HookCallbackMatcher.timeout`, `SyncHookJSONOutput` (`continue`,
`stopReason`, `decision`, `reason`, `systemMessage`), `AsyncHookJSONOutput`, and each `*HookInput` /
`*HookSpecificOutput`: `updatedToolOutput` on `PostToolUse`, `additionalContext` on `PreToolUse`, the
`PermissionRequest` `decision`, `Stop`'s `stop_hook_active` and `last_assistant_message`, `includeHookEvents`.

## Step 4: Experiment before designing

Two scratch scripts called the SDK directly (the `CLAUDE*` variables removed, Haiku, `persistSession: false`) with a
logging callback on **every** event. What shaped the design:

- Only 10 events reached a callback: `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`,
  `PostToolUseFailure`, `PostToolBatch`, `SubagentStart`, `SubagentStop`, `MessageDisplay`, `Stop`. So the lab
  shows **all** of them firing (the observer + the grid) instead of assuming.
- `SessionStart` / `SessionEnd` never fired, with a string prompt or in streaming input mode. They were dropped
  from the hooks and documented as "not delivered".
- `PermissionDenied` never fired, not even in `dontAsk` mode. A `system/permission_denied` message comes instead,
  so the UI shows that message.
- `includeHookEvents: true` added no `hook_*` messages for callbacks, so the server's own `hook` SSE event stays.
- A throwing hook is **ignored** (the call runs), while one past its `timeout` **blocks** the call. That contrast
  became scenarios 10 and 11.
- `systemMessage` arrives as `system/informational` and the model does not see it → the "shown to the user" card.
- A blocked prompt and `continue: false` on `UserPromptSubmit` give `result/success` with 0 turns and $0.

## Step 5: Design the concept

- **One SSE route**, like Concept 7. The browser sends hook names and tool names, both checked against fixed lists.
- **An observer on every event**, so a run always shows which events fired, in order, even with no hooks ticked.
- **One hook per kind of answer**, 11 in all, and 12 scenarios that each tick the hooks for one idea.
- **Compare with the disk**: `/files` and `/file` show that `updatedToolOutput` does not change `config.env`.
- `allowedTools` holds every ticked tool except `Write`, so `Write` "asks" and `PermissionRequest` fires.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/20-hooks-in-depth.ts` | New: `/run` (SSE), the hooks, `/files`, `/file`, `/reset` |
| `server/index.ts` | Mounted on `/api/c20` |
| `src/concepts/Concept20HooksInDepth.tsx` | New: scenarios, events grid, timeline, result |
| `src/App.tsx` | The tab |
| `src/styles.css` | `.events-grid`, `.ev`, `.tool-call.observer` |
| `.gitignore` | `hooks-lab/` |

`npx tsc --noEmit -p .` passed (after typing the `later()` callback's data as an object), and `npx vite build`
succeeded (the `dist/` it made was removed).

## Step 7: Test the routes

The real server was started on port **3001** with `.env` loaded and the `CLAUDE*` variables removed, and driven by a
Node script like the browser does:

| Scenario | Result |
|---|---|
| 1 · observer only | `UserPromptSubmit → MessageDisplay → PreToolUse → PostToolUse → PostToolBatch → Stop` |
| 2 · `context` | *The budget is 1,200 EUR (1,320 USD).* |
| 3 · `block-secrets` + `context` | Both hooks ran. 0 turns, $0, the block text as the result |
| 4 · `maintenance` | 0 turns, $0, `Operation stopped by hook: …` |
| 5 · `redact` | The model listed `API_KEY = [REDACTED]`. `config.env` on disk still has the key |
| 6 · `explain-failure` | `PostToolUseFailure`, then the model read `notes.txt` |
| 7 · `approve-writes` | `summary.md` created. `outside.md` in `permission_denials` with the hook's message |
| 8 · `stop-gate` | Two `Stop` calls: `block`, then `stop_hook_active: true` → `{}`. The answer ends with `Source:` |
| 9 · `subagent-brief` | *2 files: notes.txt, config.env* (after the fix below) |
| 10 · `crash` | The hook threw, the Read ran, the model printed the key |
| 11 · `slow` | Aborted after 2002 ms, twice, and the Read never ran |
| 12 · `async-log` | Returned at 1903 ms; its work finished at 3415 ms |
| empty prompt | `error` event, no query started |
| `hooks: ["rm -rf", "__proto__", "constructor"]`, `tools: ["Bash"]` | Ignored (after the fix below) |
| `/file?name=../.env` | 404 |

Two changes came from the tests:

- **Scenario 9** at first only ticked `Agent`. The `counter` subagent needs `Glob`, which was not in the session's
  pool, so the model fell back to a background general-purpose agent and the run ended before `SubagentStop`. The
  scenario now ticks `Glob` too and asks the model not to run the agent in the background.
- **Hook names** were checked with `name in available`, which also accepts `"constructor"` and `"__proto__"`. They
  were registered under an `"undefined"` event. The check is now `Object.hasOwn(available, name)`.

Costs: $0 to $0.022 per scenario on Haiku, about $0.10 for the route tests and about $0.07 for the probes.

## Step 8: Run it in the real app

Vite was started on port **5199** next to the server. It served the page, `App.tsx` with the new tab,
`Concept20HooksInDepth.tsx` (HTTP 200), and `/api/c20/files` through the Vite proxy. Both processes were stopped.
Even with `persistSession: false`, the subagent runs had left `.meta.json` files under
`~/.claude/projects/…-sample20-hooks-lab/`. That folder held only files from these tests, and it was deleted.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/20-hooks-in-depth.ts` | New: the Concept 20 routes |
| `server/index.ts` | Mounts `/api/c20` |
| `src/concepts/Concept20HooksInDepth.tsx` | New: the Hooks in depth tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Events grid and observer rows |
| `.gitignore` | Ignores `hooks-lab/` |
| `Tab1-query().md` | Adds Concept 20 to the table |
| `Tab20-Hooks-in-depth.md` | Explanation of the concept |
| `Build-steps.md` | This file |
| `readme.md` | Same content as `Tab20-Hooks-in-depth.md` |
