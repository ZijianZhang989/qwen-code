# RFC: daemon-backed Agent View for managing background sessions

## Summary

This is an RFC for adding a Claude Code-style Agent View to Qwen Code: a
TUI dashboard for dispatching, monitoring, peeking, attaching, and
backgrounding multiple daemon-backed sessions.

The recommended v1 scope is intentionally narrow:

```text
1 Agent View = current workspace daemon x N sessions
```

Qwen Code already has the main daemon foundation:

- `qwen serve`
- ACP-over-HTTP / SSE
- one daemon bound to one workspace
- multiple sessions multiplexed through one `qwen --acp` child
- daemon session list/load/resume/prompt/event APIs
- TUI event adapter for daemon sessions

The missing piece is not a new daemon. The missing piece is the interactive
TUI management layer plus one explicit background-session lifecycle so
detached sessions remain visible and attachable.

Related daemon/session work:

- #3803
- #4175
- #4514
- #5976
- #6378

## Goals

- Add a `qwen agents` style TUI dashboard for daemon-backed sessions.
- Let users dispatch a new background session from the dashboard input.
- Show sessions grouped by state: Needs input, Working, Completed/Idle.
- Let users attach to a row as a full interactive daemon session.
- Let attached daemon sessions detach back to Agent View without stopping.
- Reuse existing ACP/REST/SSE daemon infrastructure where possible.
- Preserve current daemon APIs and behavior for existing clients.
- Keep v1 scoped to the current daemon workspace.
- Make future cross-workspace aggregation possible without baking it into v1.

## Non-goals for v1

- Cross-workspace Agent View.
- Dynamic daemon discovery across all projects.
- Replacing `qwen serve` with a Claude-style supervisor process.
- One worker process per session.
- Moving an arbitrary already-running local TUI session into daemon with full
  in-flight state.
- AI-generated row summaries.
- PR status, worktree cleanup, scheduled loop rows, or advanced filters.
- Changing existing `session/detach` semantics for all clients.

## Recommended architecture

Use Agent View as an additive daemon client/dashboard.

```text
Qwen TUI
  |
  +-- AgentViewScreen
        |
        +-- AgentViewClient
              |
              +-- qwen serve, current workspace
                    |
                    +-- AcpSessionBridge
                    |     +-- session A
                    |     +-- session B
                    |     +-- session C
                    |
                    +-- agent-view roster state
```

The serve layer remains single-workspace in v1. The bridge remains the owner
of live sessions. Agent View consumes a workspace-level roster API and
per-session event streams only when peeking or attaching.

## Key design points

### 1. Background session lifecycle

Current detach behavior can close a live session when the last
client/subscriber leaves and no prompt is active. That is correct for existing
clients and should not be globally changed.

Add an explicit background lifecycle instead:

```ts
type SessionLifecycle =
  | 'attached'
  | 'background'
  | 'completed'
  | 'failed'
  | 'stopped';
```

New bridge operation:

```ts
backgroundSession(sessionId: string, clientId?: string): Promise<void>;
```

New ACP extension:

```text
_qwen/session/background
```

Optional REST equivalent:

```http
POST /session/:id/background
```

Rules:

- `backgroundSession` marks the session as background-owned.
- It then detaches the requesting client.
- Background sessions are not reaped by ordinary last-client detach.
- Existing `detachClient` and `_qwen/session/detach` keep their current
  behavior.
- Background sessions still count toward `maxSessions`.
- Non-pinned background sessions may later be subject to an idle TTL.

### 2. Workspace roster API

Agent View should not open N SSE streams just to draw the list.

Add a workspace-level roster surface:

```http
GET /agent-view/sessions
GET /agent-view/events
```

Or, if maintainers prefer extension-only first:

```text
_qwen/agent_view/list
_qwen/agent_view/subscribe
```

Recommended row model:

```ts
interface AgentViewSessionRow {
  sessionId: string;
  workspaceCwd: string;
  title?: string;
  status:
    | 'needs_input'
    | 'working'
    | 'idle'
    | 'completed'
    | 'failed'
    | 'stopped';
  summary?: string;
  createdAt: number;
  lastActivityAt: number;
  pendingPermissionCount: number;
  pendingPromptCount: number;
  hasActivePrompt: boolean;
  clientCount: number;
  subscriberCount: number;
  currentModelId?: string;
  currentApprovalMode?: string;
  pinned?: boolean;
}
```

Roster event types:

```ts
type AgentViewRosterEvent =
  | { type: 'snapshot'; sessions: AgentViewSessionRow[] }
  | { type: 'upsert'; session: AgentViewSessionRow }
  | { type: 'remove'; sessionId: string; reason: string };
```

Status derivation for v1:

- `needs_input`: pending permission or user action required.
- `working`: active prompt or queued prompt.
- `idle`: live, no prompt, no pending input.
- `completed`: latest prompt settled successfully and no more work is pending.
- `failed`: latest prompt or session ended with error.
- `stopped`: user explicitly stopped/killed the session.

### 3. Dispatch from Agent View

Every prompt submitted in the Agent View bottom input starts a new daemon
session.

Flow:

```text
input Enter
  -> session/new, sessionScope=thread
  -> session/prompt
  -> roster row appears as working
  -> user stays in Agent View
```

Do not send the prompt to the currently highlighted row in v1. Follow-up
replies belong in peek or attached mode.

### 4. Attach and detach

Attach flow:

```text
select row + Enter/Right
  -> session/load or session/resume
  -> create DaemonTuiSessionClient
  -> reuse daemon-tui-adapter
  -> render full interactive session
```

Detach flow from attached daemon session:

```text
empty input + Left
  -> _qwen/session/background
  -> return to Agent View
  -> row remains live
```

This should be limited to daemon-attached sessions in v1. Backgrounding an
arbitrary foreground non-daemon TUI session can be a later phase.

### 5. Peek and reply

Peek is a lightweight row detail panel.

v1 behavior:

- `Space` opens/closes peek.
- Shows latest relevant output, active tool summary, or pending permission.
- If permission is pending, user can approve/deny.
- If session is idle/needs input, user can send a reply via `session/prompt`.
- `Right` or `Enter` attaches from peek.

Peek can initially be backed by:

- roster row state;
- latest cached per-session events;
- on-demand `GET /session/:id/events` replay.

### 6. Existing UI relationship

Current `packages/cli/src/ui/components/agent-view/*` is an in-process agent
tab UI. It should not be treated as the daemon Agent View surface.

Recommended split:

```text
components/agent-view/        existing in-process tabs
components/daemon-agent-view/ new daemon-backed dashboard
ui/daemon/                    existing daemon adapters
```

The dashboard should reuse:

- `daemon-tui-adapter`
- existing history/tool/permission renderers where possible
- existing background task visual language where it fits

### 7. Compatibility

Existing routes and methods must keep their behavior:

- `POST /session`
- `POST /session/:id/load`
- `POST /session/:id/resume`
- `POST /session/:id/detach`
- `GET /session/:id/events`
- ACP `session/new`
- ACP `session/list`
- ACP `session/prompt`
- ACP `_qwen/session/detach`

Agent View adds new capabilities instead of changing old semantics.

Add capability tags:

```ts
agent_view
agent_view_roster
session_background
```

Older daemons should simply not show Agent View entry points.

### 8. Capacity and resource controls

- Background sessions count toward `maxSessions`.
- Roster stream should be one workspace-level SSE stream.
- Per-session SSE should be opened only for peek/attach.
- Keep existing `maxConnections`, event ring, and pending prompt caps.
- Add an optional background idle TTL later if resource retention becomes a
  problem.
- `pinned` should eventually mean keep the session process alive while idle.

## Rollout plan

### Phase 1: background lifecycle, no UI

- Add bridge-level background lifecycle flag.
- Add `backgroundSession`.
- Add `_qwen/session/background`.
- Keep current detach behavior unchanged.
- Add tests proving:
  - normal detach can still reap;
  - background detach does not reap;
  - background sessions still count toward capacity.

### Phase 2: roster API

- Add `AgentViewSessionRow` type.
- Add `GET /agent-view/sessions`.
- Add `GET /agent-view/events`.
- Publish roster events from prompt start/end, permission request/resolution,
  session close/kill, metadata update.
- Add capability tags.
- Add protocol docs.

### Phase 3: read-only Agent View TUI

- Add `qwen agents` command or `qwen --agents`.
- Render header, grouped rows, empty state, loading/error states.
- Use roster API + roster SSE.
- Support keyboard navigation and Esc exit.
- No dispatch/attach yet except maybe row selection display.

### Phase 4: dispatch new sessions

- Bottom input creates a new daemon session and sends first prompt.
- Use `sessionScope=thread`.
- Keep user in Agent View.
- Surface creation/prompt errors inline.
- Add tests for dispatch and status transition.

### Phase 5: attach/detach

- Attach row to full daemon session.
- Reuse `daemon-tui-adapter`.
- Empty input + Left calls `_qwen/session/background`.
- Return to Agent View with selected row preserved.
- Add manual E2E coverage with two sessions.

### Phase 6: peek/reply/permission

- Add peek panel.
- Show latest row activity and pending permission.
- Support approve/deny.
- Support reply to a selected session without attaching.
- Keep advanced filters and AI summaries deferred.

### Phase 7: organization and polish

- Add rename/pin/group if needed.
- Reuse existing `session_organization` where possible.
- Add notification for needs-input/completed/failed.
- Add settings:
  - `disableAgentView`
  - `defaultToAgentsView`
  - `leftArrowOpensAgents`

## Public documentation updates required

The PR that exposes the feature should update:

- `docs/users/qwen-serve.md`
  - explain daemon Agent View and current-workspace scope.
- `docs/developers/qwen-serve-protocol.md`
  - document `agent_view`, `agent_view_roster`, `session_background`.
  - document `/agent-view/sessions`, `/agent-view/events`.
- CLI reference
  - document `qwen agents`.
- Changelog
  - mention that existing detach APIs are unchanged.
  - mention current workspace limitation.

## Test plan

Focused tests should cover:

- normal detach still reaps under existing conditions;
- background detach keeps the session live;
- background session remains listed after last client leaves;
- background session capacity accounting;
- roster snapshot shape;
- roster upsert/remove events;
- prompt start/end status changes;
- permission request/resolution status changes;
- `qwen agents` empty state;
- row grouping and navigation;
- dispatch creates a new thread session;
- attach renders daemon session events;
- attached empty-left detaches back to Agent View;
- old clients using `/session/:id/detach` remain unaffected;
- capability-gated clients hide Agent View when unsupported.

## Open decision points

- Should the first public command be `qwen agents`, `qwen agent-view`, or both?
- Should Agent View use REST routes first, ACP extensions first, or both?
- Should `completed` mean "latest turn settled" or "session explicitly
  finished"?
- Should non-pinned background sessions have a TTL in v1?
- Should row metadata live in existing `session_organization` sidecar or a new
  `agent-view` sidecar?
- Should `Left` detach be enabled by default or require `leftArrowOpensAgents`?

## Requested maintainer feedback

- Please confirm v1 should be current-workspace only.
- Please confirm existing `session/detach` semantics must remain unchanged.
- Please confirm a new explicit `session_background` capability is acceptable.
- Please confirm Agent View should be built as an additive daemon dashboard
  rather than a new supervisor.
- Please confirm cross-workspace aggregation should wait for the
  multi-workspace/coordinator work.
