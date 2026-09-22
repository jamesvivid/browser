# JEV in-process agent — design

Goal: the [jev-ultrafast](https://github.com/browser-use/jev-ultrafast) decision
loop running natively inside the browser process, with [TypeSafe System One
(Jev)](https://docs.typesafe.ai) as a first-class *policy* engine next to the
existing chat providers. One typed request per step chooses the operation and
the target from the observed action space; native tools execute it with no CDP,
no WebSocket, no serialization boundary. Sub-cent per step, no external
orchestration.

Fork of lightpanda-io/browser for that work. Upstream-first: anything general
lands as PRs; this document is the map.

## Why a policy engine, not a provider

`zenai.provider.Client` is chat-shaped (messages in, text/tool-calls out).
System One is judgment-shaped: typed questions (choice / noul / score with
criteria) in, calibrated answers out. Shoehorning it behind a chat interface
would lose the property that makes it cheap and safe — answers are constrained
to the offered options. So JEV sits beside the provider layer as a *strategy*:

```
lightpanda agent --policy jev --task "Open the top story's discussion on news.ycombinator.com"
```

`--policy llm` (default) keeps today's behavior untouched. TYPE_TEXT field
values still route to a chat provider (the choose/generate split is preserved:
the judgment model never writes prose).

## The loop (ported from the validated external version)

Per step, in-process:

1. **Observe.** `lp.interactive.collectInteractiveElements` (tag, role,
   accessible name, value, interactivity type) + `registerNodes` → backendNodeIds,
   plus page URL/title/text. This is the element table; it already exists.
2. **Decide.** One HTTPS POST to `https://api.typesafe.ai/v1/systemone` with the
   question set validated in the external POC: one `choice` question over the
   operations (`CLICK`, `TYPE_TEXT`, `SELECT`, `SCROLL_*`, `WAIT`, `DONE`,
   `BLOCKED`) plus speculative `choice` target heads per operation, criteria
   keyed by backendNodeId. One network round trip per step.
3. **Execute.** Native tool by id: `click`/`fill`/`selectOption`/`setChecked`
   (backendNodeId form), `scroll`, or nothing for `WAIT`/`DONE`/`BLOCKED`.
   JEV never emits selectors — ids come from the observed table only.
4. **Settle.** Existing `waitForState` / short settle, then back to 1.

Budgets and honesty port unchanged: action cap, request cap, `BLOCKED`
propagates as a terminal answer, `DONE` requires the task's own verification
step (a `waitForScript`/`extract` check supplied with the task), never the
model's word alone.

## Files

| Path | Role |
| --- | --- |
| `src/agent/jev/Policy.zig` (new) | question construction, response validation, action-space assembly |
| `src/agent/jev/Client.zig` (new) | System One HTTP client (key from `TYPESAFE_API_KEY`, mirroring zenai client patterns: retry, timeout) |
| `src/agent/Agent.zig` | `--policy jev` branch: run the loop instead of the chat turn for `--task` |
| `src/Config.zig` | `--policy` flag; `TYPESAFE_API_KEY` in the env-key list |
| `src/browser/tools.zig` | unchanged — consumed, not modified |

## Measurement (the point of the exercise)

Same task, three arms, reported side by side:

1. `lightpanda agent --task ...` (LLM provider, today's path)
2. external jev-ultrafast loop over CDP (the validated POC numbers)
3. `lightpanda agent --policy jev --task ...` (this work)

Cost per step, wall time, requests per task, completion. External POC baseline
to beat on overhead: 29 CDP calls and ~63 ms of protocol time per task; Jev
latency 223–800 ms per request dominates — the in-process version removes the
protocol slice entirely and should approach pure model latency + native tool
cost (µs).

## Boundaries

- No browser impersonation anywhere (upstream policy is the product).
- `TYPESAFE_API_KEY` stays in env/config, never recorded in PandaScripts.
- Replay (PandaScript) stays deterministic and token-free: JEV decides at
  prototype time; replays don't call it.
