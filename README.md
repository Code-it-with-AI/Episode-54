# Episode-54: MCP v2 Tasks Extension

📺 YouTube video: https://youtu.be/

🏠 Code it with AI Home Page: https://codeitwithai.com

## What is the Tasks extension?

MCP version 2 (spec revision `2026-07-28`) is stateless by default: there's no `initialize` handshake and no `Mcp-Session-Id`, so every request stands on its own. That works well for quick tool calls, but not for a tool that takes a long time to finish (say, 40 seconds, or 40 minutes).

The **Tasks extension** (`io.modelcontextprotocol/tasks`) solves this. Instead of making the client wait for a result, the server answers `tools/call` with a **task handle**, and the client checks back for the result later. The handle is durable, so the client can disconnect, crash, or restart and still pick the same task up again, without a second `tools/call`.

Tasks complement **multi round-trip requests (MRTR)**, the other MCP v2 dialog feature:

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
- **Polling, not push (for now).** The spec also defines a `notifications/tasks` push notification, but the MCP C# SDK 2.2.0 doesn't implement it yet, so clients poll.

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
