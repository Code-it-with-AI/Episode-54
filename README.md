# Episode-54: MCP v2 Tasks Extension

📺 YouTube video: https://youtu.be/ECPto-Pi2co

🏠 Code it with AI Home Page: https://codeitwithai.com

## What is the Tasks extension?

MCP version 2 (spec revision `2026-07-28`) is stateless by default: there's no `initialize` handshake and no `Mcp-Session-Id`, so every request stands on its own. That works well for quick tool calls, but not for a tool that takes a long time to finish (say, 40 seconds, or 40 minutes).

The **Tasks extension** (`io.modelcontextprotocol/tasks`) solves this. Instead of making the client wait for a result, the server answers `tools/call` with a **task handle**, and the client checks back for the result later. The handle is durable, so the client can disconnect, crash, or restart and still pick the same task up again, without a second `tools/call`.

### Building on Episode-50: elicitation and MRTR

In [Episode-50: MCP 2 Dialogs with Multi Round-Trip Requests](https://github.com/Code-it-with-AI/Episode-50) ([video](https://youtu.be/WDH9zguuyKg)), a server asks the user follow-up questions in the middle of a tool call. It uses elicitation and **multi round-trip requests (MRTR)**, with the encrypted `requestState` token taking the place of a session. That's near-realtime messaging: each round is a quick request and response, while a person answers questions on the other end.

The Tasks extension carries that same dialog over to **long-running work**. The server can still ask a question (the task goes to `input_required`), but the question shows up when the client polls, and the work keeps running while the user is gone. Here's how the two compare:

| | MRTR | Tasks |
|---|---|---|
| Use it when | The server needs more information *before* it can answer | The work takes a long time to run |
| The server returns | `resultType: "input_required"` with `inputRequests` and a `requestState` | `resultType: "task"` with a `taskId` |
| The client then | Re-sends the same `tools/call` with `inputResponses` and the echoed `requestState` | Polls `tasks/get` until the task finishes |
| State lives in | The opaque `requestState` that the client echoes back | The server's task store |

## How it works

```
Client                                                         Server
  │  tools/call book_freight                                     │
  │    _meta: clientCapabilities.extensions                      │
  │           ["io.modelcontextprotocol/tasks"]                  │
  │ ───────────────────────────────────────────────────────────► │  task stored, work starts in the background
  │  ◄── resultType: "task"  taskId, status: working,            │
  │                          ttlMs, pollIntervalMs               │
  │                                                              │
  │  tasks/get {taskId}               (every pollIntervalMs)     │
  │  ◄── working  "Reserving carrier capacity (2/5)"             │
  │                                                              │
  │  ✂ client goes away … comes back with the same taskId        │  …still working
  │                                                              │
  │  tasks/get {taskId}                                          │
  │  ◄── input_required  inputRequests: { <key>: elicitation }   │  the work asks a question
  │  tasks/update {taskId, inputResponses: { <key>: ... }}       │
  │  ◄── {} (ack)                                                │
  │                                                              │
  │  tasks/get {taskId}                                          │
  │  ◄── completed  result: { content: [...] }                   │  same result tools/call would have returned
```

### Key concepts

- **Opt-in per request.** The client declares the extension in each request's `_meta` (under `io.modelcontextprotocol/clientCapabilities` → `extensions`). The server advertises support in `server/discover`. The server decides, call by call, whether to return a task, and it must never return one to a client that didn't declare the extension.
- **The task handle.** A `tools/call` that becomes a task returns a `taskId`, a `status`, an optional `statusMessage`, `createdAt` and `lastUpdatedAt` timestamps, a `ttlMs` (how long the server keeps the task), and a suggested `pollIntervalMs`.
- **Task lifecycle.** A task is `working`, `input_required`, `completed`, `failed`, or `cancelled`. A `completed` task carries the same result that a normal `tools/call` would have returned.
- **The `tasks/*` methods:**
  - `tasks/get` returns the current status and, once the task is finished, its result.
  - `tasks/update` sends `inputResponses` back when a task is `input_required`.
  - `tasks/cancel` asks the server to stop the work.
- **Questions inside a task.** While the work is running, the server can ask for input, such as an elicitation or a sampling request. The task moves to `input_required` with an `inputRequests` map. The client answers with `tasks/update`, using the same keys, and the task goes back to `working`. Keys are unique for the whole life of the task, so the client can de-duplicate them across polls.
- **Durability is up to the server.** A handle is only as durable as the server's task store. With an in-memory store, a server restart loses the task. With a persistent store, the work can carry on after a restart. When the TTL runs out, the server can delete the task, and an unknown or expired `taskId` gets error **-32602**.
- **Load balancing.** Over Streamable HTTP, the client sets the `Mcp-Name` header to the `taskId` on `tasks/*` requests, so a gateway can route polls to the right place.
- **Errors.** A client that calls `tasks/*` without declaring the extension gets **-32021**. A server that can't serve a non-declaring client without a task must also return -32021. A server that can serve it in another way (for example, with a plain-text answer) returns an ordinary result.
- **Polling, not push (for now).** The spec also defines a `notifications/tasks` push notification that is delivered over `subscriptions/listen` (see [Events](#events-when-you-dont-need-a-task)), but the MCP C# SDK 2.2.0 doesn't implement it yet, so clients poll.

## The demo

The demo code is in the [`demo/tasks` branch of MarimerLLC/Mcp2-Dialog](https://github.com/MarimerLLC/Mcp2-Dialog/tree/demo/tasks). It builds on the MRTR shipping-quote sample and adds a slow `book_freight` tool:

- The tool takes about 40 seconds and runs in five named stages, updating its `statusMessage` as it goes.
- About halfway through, the carrier "moves the pickup window", and the task goes to `input_required` until the user answers.
- The CLI agent saves the `taskId` before its first poll. You can kill it part-way (Ctrl+C or `Stop-Process -Force`) and run `book --resume` to carry on with the same task. The summary shows **one `tools/call` across both agent runs**.
- The agent saves the trace context with the task id, so in the Aspire dashboard the killed run and the resumed run are **one trace**.
- An optional file-backed task store lets unfinished bookings carry on after a server restart.

It uses the official [MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) 2.2.0 and its `ModelContextProtocol.Extensions.Tasks` package:

- **Server:** `WithTasks(store, options)` registers the `tasks/*` handlers, advertises the extension, and runs task-backed tools in the background. Calls to `ElicitAsync` inside a task become `inputRequests` automatically.
- **Client:** `CallToolAsTaskAsync` handles both result shapes (a direct result or a task handle). `GetTaskAsync`, `UpdateTaskAsync`, and `CancelTaskAsync` cover the rest. `CallToolWithPollingAsync` runs the whole poll loop if you don't need to resume.

The demo adds a few things that SDK 2.2.0 doesn't have yet: tools updating their own `statusMessage`, a way for a tool to learn its own task id, and resuming tasks after a server restart. One limit to know about: SDK 2.2.0 can't combine MRTR with a task in the same tool, so `book_freight` asks its mid-flight question with `ElicitAsync` instead.

## Events: when you don't need a task

A task is for one tool call whose *server-side work* takes a long time. Often nothing is running on the server at all. The agent just wants to hear when something changes: people voting on a meeting poll over several days, a file being edited, or the server's tool list changing. You could model that as a task that completes on the next change, but it's an expensive fit:
- The server keeps a background handler running, and a restart kills it.
- Every timeout completes the task even when nothing happened.
- Each watched item uses up a task.

For this case MCP v2 has a separate, lighter mechanism: the **`subscriptions/listen`** request ([SEP-2575](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/seps/2575-stateless-mcp.md), [spec](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/patterns/subscriptions.mdx)). The server sends **events** (JSON-RPC notifications), and the client **listens** for them.

### How listening works

```
Client                                                          Server (stateless)
  │  subscriptions/listen                                         │
  │    notifications: { resourceSubscriptions:                    │
  │                       ["scheduler://events/abc/status"],      │
  │                     toolsListChanged: true }                  │
  │ ────────────────────────────────────────────────────────────► │  the response (SSE) stays open
  │  ◄── notifications/subscriptions/acknowledged                 │  only what the server agreed to
  │        { resourceSubscriptions: [".../abc/status"] }          │
  │                                                               │
  │        … silence: nothing changes, nothing is sent …          │
  │                                                               │
  │  ◄── notifications/resources/updated { uri: ".../abc/status" }│  something changed
  │  resources/read ".../abc/status"     (an ordinary request)    │
  │  ◄── { contents: [...] }                                      │
  │                                                               │
  │  ◄── { resultType: "complete" }      server ends the stream   │
  │  subscriptions/listen …  (client re-subscribes and re-reads)  │
```

- **It replaces the v1 plumbing.** In MCP v1, notifications came over a session's standalone HTTP GET stream, and clients used `resources/subscribe` / `resources/unsubscribe`. MCP v2 has no sessions, so all three are gone. Instead, the client opens one long-lived `subscriptions/listen` request, and the server sends notifications on that request's own response stream.
- **The client says what it wants.** The `notifications` filter has four optional fields: `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, and `resourceSubscriptions` (a list of resource URIs). The server must not send any type the client didn't ask for.
- **The server says what it agreed to.** Its first message must be `notifications/subscriptions/acknowledged`, listing the subset it will honor. The client should compare that with what it asked for.
- **Correlation.** Every message on the stream carries `_meta["io.modelcontextprotocol/subscriptionId"]`, which is the JSON-RPC id of the listen request. A client can have several listen streams open at once.
- **Events are signals, not data.** `notifications/resources/updated` carries only the URI. The client reads the resource again with `resources/read`, so the content always goes through the server's normal access checks.
- **Capabilities.** A server declares `resources.subscribe` if it supports per-resource updates, and `listChanged` on tools, prompts, or resources if it sends those notifications.
- **What doesn't go on the listen stream:**
  - Progress (`notifications/progress`) and log messages (`notifications/message`) belong to the request they're about. They go on that request's own response stream, for example while a `tools/call` is running.
  - Server-to-client *requests*, such as elicitation or sampling, are never sent on the listen stream. Those use MRTR, as in Episode-50.
- **Ending and reconnecting.** The client ends a subscription by closing the stream. The server can end it too. It should send a final `resultType: "complete"` response first, and the client can treat any other disconnect as a reason to reconnect. There's no `Last-Event-ID` resumption, and the server keeps no subscription state. So after reconnecting, the client sends its full listen request again and re-reads what it watches, to catch anything it missed.
- **Keep-alives.** The spec encourages servers to send SSE comment lines (`:`) on a quiet stream, so proxies don't cut it.
- **Tasks can use it too.** The Tasks extension adds `taskIds` to the filter and a `notifications/tasks` event, so a client could be told when a task changes instead of polling. The MCP C# SDK 2.2.0 doesn't implement this yet.

### Tasks or events?

| | Task | Event subscription |
|---|---|---|
| The slow thing is | Work the server is doing for one tool call | Something that changes on its own schedule (people, other systems) |
| Server state | A durable task store, with the task recorded before the handle is returned | None: the subscription *is* the open stream |
| When nothing happens | The client keeps polling, and the task eventually hits its TTL or the client's timeout | Silence, at no cost |
| Watching ten things | Ten tasks | One stream listing ten URIs |
| What the client gets | The tool's result, from `tasks/get` | A "this changed" signal; the client reads the resource |
| Asking the user something | `input_required` and `tasks/update` | Not on the stream; use MRTR on ordinary tool calls |

**Rule of thumb:**
- The server needs an answer *now* → MRTR (Episode-50).
- The server is *doing* something that takes a long time → a task.
- The server is *waiting* for the world to change → expose the thing as a resource and let clients subscribe to it.

### A real example: Scheduler and RockBot

[Scheduler](https://github.com/MarimerLLC/scheduler) is an MCP server that coordinates meeting times: the organizer's agent proposes slots and each invitee's agent votes. The organizer's agent wants to know when everyone has answered. The first plan was a `wait_for_update` task. On 3 November 2026 it was replaced with resource subscriptions, for the reasons above. The details are in [docs/design.md, "Push updates: subscriptions instead of tasks"](https://github.com/MarimerLLC/scheduler/blob/main/docs/design.md#push-updates-subscriptions-instead-of-tasks).

**Server side (Scheduler, MCP C# SDK 2.2.0, stateless Streamable HTTP):**

- **Each event is a resource.** `scheduler://events/{shortCode}` returns what `get_event` returns, with the same access check.
- **A coarser status resource.** `scheduler://events/{shortCode}/status` changes only at milestones: everyone required has voted, someone rules out every time, the deadline passes, or the event is finalized, reopened, or cancelled. Watching it means one wake-up per milestone instead of one per vote. It includes a deterministic `marker` hash, so a client can tell whether anything changed, even across reconnects.
- **A custom `subscriptions/listen` handler** (`EventSubscriptions`, plugged in as the SDK's `SubscriptionsListenHandler`):
  - It acknowledges only the URIs the caller is allowed to read, and ignores `*ListChanged` requests.
  - It forwards changes as `notifications/resources/updated`, sending a status URI only when its marker moves.
  - It caps each caller at 5 streams that watch something, 20 idle streams, and 50 URIs per stream.
- **No keep-alives; the stream ends instead.** The SDK has no API for keep-alives on a request stream, so the server ends each stream after 25 minutes and the client reconnects. This also limits how long a stream can outlive its bearer token, which is only checked when the stream opens.
- **Stores nothing.** After a restart or deploy, clients reconnect, subscribe again, and re-read.
- **Push is optional.** Clients that don't listen keep working with tools and email. Clients on an older protocol revision get method-not-found for `subscriptions/listen`.
- **Claude Code opens a listen stream too.** It opens one on every connection, asking only for `list_changed`. Scheduler keeps that stream open, idle, rather than closing it, because clients retry as soon as a stream ends.

**Client side ([RockBot](https://github.com/MarimerLLC/rockbot), planned in [rockbot#768](https://github.com/MarimerLLC/rockbot/issues/768)):** this is the "agent listens for events" half. The agent doesn't sit in a loop waiting. RockBot's MCP bridge holds the stream and wakes the agent when something happens.

- **One stream per server.** The bridge keeps one `subscriptions/listen` stream open for each server the operator opts in, covering every watched URI.
  - It reconnects with back-off, and also when the token is refreshed.
  - After any reconnect, including an agent restart, it re-reads each watched resource and compares a content hash to catch missed changes.
- **Watching from a conversation.** The agent gets `mcp_watch_resource` / `mcp_unwatch_resource` tools, bound to the conversation that called them. Watches are kept in a ledger, so they survive restarts.
- **Updates become turns.** On `notifications/resources/updated`, the bridge reads the resource and gives the conversation a new turn. Bursts of updates are debounced into one turn, and the content is treated as untrusted, server-written data.
- **Why not RockBot's task support:** a task has to end at a timeout, and every "nothing changed" completion wastes an agent turn. Each watched item would also hold one of the 3 task slots a conversation gets. A subscription is silent until something changes, and one stream covers everything watched on a server.
