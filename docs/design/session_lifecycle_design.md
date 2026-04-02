---
codd:
  node_id: design:session-lifecycle-design
  type: design
  depends_on:
  - id: design:system-design
    relation: depends_on
    semantic: technical
  - id: design:ipc-hook-design
    relation: depends_on
    semantic: technical
  depended_by:
  - id: design:chat-rendering-design
    relation: depends_on
    semantic: technical
  - id: detail:session_state_machine
    relation: depends_on
    semantic: technical
  - id: detail:ipc_message_flow
    relation: depends_on
    semantic: technical
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:session-lifecycle-design
    reason: Session state updates received from the Unix socket must be reflected
      in UI within 100 ms (REQ-PERF-02). This is a release-blocking latency constraint.
  - targets:
    - design:session-lifecycle-design
    reason: All SessionPhase transitions (idle → processing → waitingForInput → ended)
      must be covered by unit tests (REQ-TEST-02). Untested state machines block release.
  - targets:
    - design:session-lifecycle-design
    reason: Approval responses must be routed to the correct tmux pane via session
      matching (REQ-PERM-05, REQ-INT-04, REQ-INT-05). Incorrect routing is a release-blocking
      correctness failure.
---

# Session Lifecycle and Monitoring Design

## 1. Overview

This document specifies the session lifecycle model for Claude Island: how individual Claude Code CLI sessions are created, tracked through phase transitions, monitored for concurrent activity, and torn down — and how every state change is reflected in the SwiftUI overlay within the latency budget required for release.

Claude Island surfaces session state from the Unix domain socket at `~/.claude/claude-island.sock`. Hook scripts installed into `~/.claude/hooks/` emit newline-delimited JSON events for session start, tool call, tool result, `AskUserQuestion` invocation, and session end. `SessionStore` — a main-actor `ObservableObject` — consumes those events and drives the overlay through a formally defined `SessionPhase` state machine. The monitoring subsystem must support at least three concurrent sessions (AC-MON-01), detect new processes within two seconds of launch (AC-MON-03), and reflect socket events in the UI within 100 ms of socket write (REQ-PERF-02, FC-PERF-01). These are release-blocking constraints; each is enforced by a specific test requirement or failure criterion.

**Release-blocking constraints owned by this document:**

| ID | Constraint | Enforcement |
|---|---|---|
| REQ-PERF-02 / FC-PERF-01 | Socket event → `@Published` UI update ≤ 100 ms (±10 ms tolerance) | `testLatencySocketToPublished` in `ClaudeIslandIntegrationTests` |
| REQ-TEST-02 / FC-TEST-01 | All five `SessionPhase` transitions covered by passing unit tests | Unit test target in `ClaudeIsland.xcodeproj`; absence blocks release |
| REQ-PERM-05 / FC-PERM-01 | Approval responses routed to correct tmux pane via session matching | `testConcurrentTwoPaneSessions` integration test; misrouting blocks release |
| AC-MON-01 | ≥ 3 concurrent sessions monitored simultaneously | Integration test with three simultaneous hook clients |
| AC-MON-03 | New session detected within 2 s of Claude Code process launch | Latency assertion in unit/integration tests |
| AC-MON-05 | `/clear` command clears corresponding session within 100 ms | Unit test asserting `SessionStore` removes session on `session_end` within 100 ms |

---

## 2. Architecture

### 2.1 Session Data Model

`Session` is a value type (`struct`) stored in `SessionStore`. Each session is immutable between updates; `SessionStore` replaces entries in its dictionary rather than mutating them, consistent with the project's immutability requirement.

```swift
struct Session: Identifiable {
    let id: UUID                              // Matches session_id from hook event
    var phase: SessionPhase                   // Current lifecycle phase
    var title: String                         // Derived from working_dir basename
    var workingDir: URL                       // Absolute path from session_start event
    var elapsedTime: TimeInterval             // Increments via Timer while processing/waitingForInput
    var pendingPermission: PermissionRequest? // Non-nil during waitingForInput on tool_call
    var tmuxPane: String?                     // $TMUX_PANE value, e.g. "%3"; nil if not in tmux
    var chatHistoryURL: URL?                  // Path to JSONL file for this session
    var promptText: String?                   // Non-nil during waitingForInput on ask_user
}
```

`SessionStore` holds sessions in `[UUID: Session]`. Keying by `UUID` (parsed from the `session_id` string in each event) gives O(1) lookup during event dispatch and during `PermissionRouter` resolution.

### 2.2 SessionPhase State Machine

`SessionPhase` is an enum with four cases:

```swift
enum SessionPhase: Equatable {
    case idle
    case processing
    case waitingForInput
    case ended
}
```

**Valid transitions:**

| From | To | Trigger event |
|---|---|---|
| `idle` | `processing` | `tool_call` event received for session |
| `processing` | `waitingForInput` | `tool_call` or `ask_user` awaiting user action |
| `waitingForInput` | `processing` | User approves or denies; session continues |
| `processing` | `ended` | `session_end` event received |
| `waitingForInput` | `ended` | `session_end` received while awaiting input |
| `idle` | `ended` | `session_end` received before any tool call |

These are the five transitions that must be covered by passing unit tests (REQ-TEST-02, AC-TEST-02). Absence of any single test case is FC-TEST-01, a release-blocking failure.

**Invalid transition handling:**

Any transition not in the table above — for example `ended → processing`, `ended → waitingForInput`, or `waitingForInput → idle` — is rejected by `SessionStore.applyTransition(_:to:)`. Rejection emits an `os_log` error message at the `.error` level (subsystem `com.claudeisland.session`, category `state-machine`) and does not crash. The session's current phase is left unchanged. Unit tests assert that each invalid transition leaves phase unchanged and emits the expected log entry.

**Animated crab icon:**

The crab icon in `NotchOverlayView` animates during `processing` and `waitingForInput`. It halts (static) during `idle` and `ended` (AC-UI-05). SwiftUI `withAnimation(.spring(response: 0.3, dampingFraction: 0.7))` governs the halt/animate transition, targeting ≥ 60 fps (REQ-PERF-01, FC-PERF-02).

**Elapsed time timer:**

A `Timer` (scheduled on the main run loop) increments `Session.elapsedTime` once per second while `phase == .processing` or `phase == .waitingForInput`. The timer is invalidated on transition to `ended` or `idle`. Because `Session` is a value type, each timer tick produces a new `Session` value replacing the old entry in `SessionStore`'s dictionary; the `@Published` dictionary triggers a SwiftUI update.

### 2.3 SessionStore — Main-Actor ObservableObject

`SessionStore` is declared `@MainActor` and conforms to `ObservableObject`:

```swift
@MainActor
final class SessionStore: ObservableObject {
    @Published private(set) var sessions: [UUID: Session] = [:]
    // ...
}
```

All mutations to `sessions` occur on the main actor. Event dispatch from `SocketActor` (background actor) to `SessionStore` uses a Swift `async` task that bridges actors:

```swift
// Inside SocketActor, after parsing IPCEvent:
Task { @MainActor in
    sessionStore.apply(event: parsedEvent)
}
```

This pattern ensures the main thread is never blocked by socket I/O or JSON parsing, satisfying REQ-PERF-03 (JSONL parse must not block main thread) and the latency requirement that `@Published` updates occur within 100 ms of socket write (REQ-PERF-02).

The end-to-end latency path is:

```
Hook script write() → SocketActor read/parse → Task @MainActor → SessionStore.apply() → @Published update → SwiftUI diff → screen
```

Each step is non-blocking. `MessageFramer` buffers bytes on `SocketActor` until a `\n` byte is found. JSON parsing (`JSONDecoder`) runs synchronously on `SocketActor` before the main-actor dispatch. The main-actor work is limited to a dictionary update and `objectWillChange` notification.

### 2.4 Event Processing Pipeline

`SessionStore.apply(event:)` is the single entry point for all `IPCEvent` values delivered from `SocketActor`. Its dispatch table:

**`session_start`:**
1. Parse `session_id` as `UUID`; parse `working_dir` as `URL`; parse optional `tmux_pane` as `String`.
2. Create a new `Session(id:, phase: .idle, title: workingDir.lastPathComponent, workingDir:, elapsedTime: 0, tmuxPane:)`.
3. Insert into `sessions[uuid] = newSession`.
4. New session must be reflected in `SessionListView` within 2 seconds of the hook script executing (AC-MON-03). Because `session_start` is the first event emitted by the hook script, and socket-to-`@Published` latency is ≤ 100 ms, the 2-second budget is comfortably satisfied absent pathological system load.
5. Fire `AnalyticsClient.track(.session_started)` — no session content, path, or PII is included (REQ-SEC-05).

**`tool_call`:**
1. Look up session by `session_id`.
2. Validate transition: current phase must be `idle` or `processing`. If already `waitingForInput`, log an `os_log` warning and enqueue the request for sequential processing.
3. Transition phase to `waitingForInput`.
4. Populate `pendingPermission = PermissionRequest(toolName:, toolInput:, requestId:)`.
5. If `phase` was `idle`, start elapsed time timer.
6. `PermissionRouter.register(requestId: fd:)` retains the open `FileDescriptor` in its open-connection registry keyed by `request_id`.

**`ask_user`:**
1. Look up session by `session_id`.
2. Transition phase to `waitingForInput`.
3. Set `promptText` from `prompt_text` field.
4. Register open connection in `PermissionRouter` registry.
5. Activate amber indicator and overlay bounce (AC-UI-06, AC-UI-08).

**`tool_result`:**
1. Look up session by `session_id`.
2. Transition phase from `waitingForInput` to `processing`.
3. Clear `pendingPermission` and `promptText`.
4. Update elapsed time timer (already running; no restart needed).

**`session_end`:**
1. Look up session by `session_id`.
2. Transition phase to `ended` (valid from `idle`, `processing`, or `waitingForInput`).
3. Invalidate elapsed time timer.
4. If `phase` was `waitingForInput`, expire any open connection in `PermissionRouter` for this session's pending `request_id` with a deny response.
5. Play completion sound via `SoundPlayer` if the hosting terminal window is not in focus (AC-SOUND-01, AC-SOUND-05).
6. Show green checkmark in overlay for 3 seconds, then transition overlay to closed state (as specified in `design:system-design` §2.3).
7. Session entry remains in `sessions` for 3 seconds to allow the checkmark animation, then is removed. Removal triggers `@Published`, which updates `SessionListView`.
8. The `/clear` terminal command causes the Claude Code process to exit and emit `session_end`; `SessionStore` removes the session entry within 100 ms of event receipt (AC-MON-05).

### 2.5 Concurrent Session Management

`SessionStore` manages all active sessions in a single `[UUID: Session]` dictionary. The architecture imposes no cap on concurrent sessions; the acceptance criterion requires support for at least three simultaneous sessions (AC-MON-01).

`SessionListView` renders a list of active sessions derived from `sessions.values.sorted(by: { $0.elapsedTime > $1.elapsedTime })`. Switching between sessions in `SessionListView` changes the `selectedSessionID: UUID?` binding without restarting any timers or re-parsing chat history.

`SocketActor` maintains a separate per-connection buffer for each accepted file descriptor. Concurrent event processing for distinct sessions is fully parallel on `SocketActor`; each connection's frame buffer is independent. Dispatch to `SessionStore` is serialized by the main actor, but individual `apply(event:)` calls are O(1) dictionary operations and do not block.

**Stale session cleanup:** Sessions whose last event was `session_end` are removed from the dictionary after the 3-second checkmark display. Sessions for which no `session_end` event was received and whose corresponding process no longer appears in the process list are detected by a `Timer`-driven reaper that runs every 30 seconds; orphaned sessions (no corresponding PID) are transitioned to `ended` synthetically, logged at `os_log` `.info`, and removed after 3 seconds.

### 2.6 PermissionRouter and tmux Pane Routing

`PermissionRouter` is a class owned by `SessionStore` that manages the open-connection registry and delivers approval responses to the correct connection. It is accessed only from the main actor.

**Open-connection registry:**

```swift
private var openConnections: [String: FileDescriptor] = [:]  // keyed by request_id
```

On `tool_call` or `ask_user` event receipt, `SocketActor` passes the raw `FileDescriptor` of the accepting connection to `SessionStore` alongside the parsed event. `SessionStore` calls `PermissionRouter.register(requestId: String, fd: FileDescriptor)` which stores the descriptor in `openConnections`.

**User action → response write:**

When the user taps Allow or Deny in `PermissionRequestView`:

1. `PermissionRouter.respond(requestId: decision: responseText:)` is called on the main actor.
2. It looks up `openConnections[requestId]` to obtain the `FileDescriptor`.
3. It serializes the response JSON: `{"request_id":"<uuid>","decision":"allow"|"deny","response_text":"<text or null>"}`.
4. It dispatches a write task to `SocketActor` which calls `write()` on the descriptor.
5. It removes the entry from `openConnections` and closes the descriptor.

**tmux pane routing (REQ-PERM-05, REQ-INT-04, REQ-INT-05, FC-PERM-01):**

When a `tool_call` event arrives with a `tmux_pane` value (e.g., `"%3"`), `Session.tmuxPane` is populated. `PermissionRouter` does not use `tmux send-keys` to deliver the response; instead, response delivery occurs on the already-open file descriptor from the hook script that is still blocking on `read()`. This is the canonical routing mechanism: the hook script in pane `%3` opened the connection and is waiting; writing the response JSON to that connection's descriptor delivers it to pane `%3` exclusively.

The routing correctness guarantee is therefore:

- One connection per `tool_call` event, originating from the hook script process in the specific tmux pane.
- `PermissionRouter` writes only to the descriptor from which the event arrived.
- No broadcast; no `tmux send-keys` side channel; no reliance on pane ID for lookup beyond the registry entry that already contains the correct descriptor.

Misrouting is structurally impossible when the registry key is the `request_id` (a UUID generated per request) and the descriptor is the one accepted from the originating hook client. The release-blocking failure criterion FC-PERM-01 is satisfied by `testConcurrentTwoPaneSessions` in `ClaudeIslandIntegrationTests`, which runs two simultaneous hook clients each sending `tool_call` events with distinct `request_id` and `tmux_pane` values, and asserts each client receives only its own response.

**Connection expiry:**

Entries in `openConnections` older than 120 seconds are expired by a reaper `Timer` that runs every 10 seconds. On expiry, `PermissionRouter` writes a deny response (`"decision":"deny"`) to the descriptor, closes it, removes the registry entry, and emits an `os_log` warning. `SessionStore` transitions the corresponding session from `waitingForInput` to `processing` (signalling the denial) and clears `pendingPermission`. The overlay dismisses the permission request UI and shows a "Request expired" banner for 2 seconds.

### 2.7 Chat History Integration

`ChatHistoryParser` reads JSONL files from the Claude Code session directory identified by `Session.chatHistoryURL`. Parsing runs on a dedicated `ChatActor` (background actor) and is never called from the main thread, satisfying REQ-PERF-03 (FC-PERF-03 is release-blocking).

Parsed `ChatMessage` arrays are cached in `SessionStore` under `chatHistory: [UUID: [ChatMessage]]`. Switching sessions in `SessionListView` reads from the cache; no re-parse occurs (AC-CHAT-05). Cache invalidation is triggered by each `tool_result` event, which may append new assistant and tool-result messages to the JSONL file.

A corrupt or missing JSONL file produces `ChatHistoryResult.empty` or `ChatHistoryResult.error(String)` without crashing (FC-REL-02). `ChatHistoryView` renders an appropriate empty state or error message in place of history content.

### 2.8 Latency Budget — Socket to UI (REQ-PERF-02)

The 100 ms budget is allocated across the pipeline stages:

| Stage | Budget allocation | Notes |
|---|---|---|
| `SocketActor` read + frame assembly | ≤ 20 ms | Buffered read; typically < 1 ms |
| JSON decode (`JSONDecoder`) | ≤ 10 ms | Single-frame decode; typically < 1 ms |
| Main-actor dispatch (Swift async Task) | ≤ 30 ms | Depends on main run loop contention |
| `SessionStore.apply()` + dictionary update | ≤ 10 ms | O(1) dictionary op |
| SwiftUI `objectWillChange` → diff → render | ≤ 30 ms | One frame at 60 fps ≈ 16.7 ms |
| **Total** | **≤ 100 ms** | Tolerance ±10 ms |

Verification is performed by `testLatencySocketToPublished` in `ClaudeIslandIntegrationTests`: a test client writes a `session_start` frame to the real `UnixSocketServer` bound at a temporary path under `/tmp/`; a `Task` on the main actor awaits the `@Published` update and records the elapsed wall-clock time; the test asserts the interval is ≤ 110 ms (100 ms + 10 ms tolerance).

### 2.9 Unit Test Requirements — SessionPhase Transitions (REQ-TEST-02)

All five valid `SessionPhase` transitions must have passing unit tests in the `ClaudeIslandTests` target within `ClaudeIsland.xcodeproj`. The required test cases:

| Test method | Transition verified |
|---|---|
| `testIdleToProcessingOnToolCall` | `idle → processing` |
| `testProcessingToWaitingOnPermissionRequest` | `processing → waitingForInput` |
| `testWaitingToProcessingOnApproval` | `waitingForInput → processing` |
| `testProcessingToEndedOnSessionEnd` | `processing → ended` |
| `testWaitingToEndedOnSessionEnd` | `waitingForInput → ended` |
| `testIdleToEndedOnImmediateSessionEnd` | `idle → ended` |
| `testInvalidTransitionEndedToProcessingRejected` | `ended → processing` is rejected; phase unchanged |
| `testInvalidTransitionEndedToWaitingRejected` | `ended → waitingForInput` is rejected; phase unchanged |

The first five tests satisfy REQ-TEST-02 and AC-TEST-02. The invalid-transition tests are required by the `os_log`-and-no-crash invariant. Absence of the first five constitutes FC-TEST-01. All tests run against the real `SessionStore` implementation; no stub or mock `SessionStore` is used for state-machine tests.

### 2.10 Sound and Overlay Feedback on Phase Transitions

**`ended` phase with success:**
- `SoundPlayer` queries `NSWorkspace.shared.frontmostApplication` and `AXUIElementCopyAttributeValue` to determine whether the terminal window hosting the completed session has keyboard focus.
- If terminal is not in focus: `SoundPlayer` plays the user-selected system sound via `AVAudioPlayer` initialized with the sound file from `/System/Library/Sounds/` (AC-SOUND-01, AC-SOUND-05).
- `NotchOverlayView` displays a green checkmark for 3 seconds using a `withAnimation(.spring())` transition, then auto-closes (AC-UI-05 by implication of `ended` state).

**`waitingForInput` phase (permission or ask_user):**
- Amber indicator activates in `NotchOverlayView` (AC-UI-06).
- If the terminal window does not have keyboard focus, the overlay performs a bounce animation to attract attention (AC-UI-08).
- The bounce is a `withAnimation(.interpolatingSpring(stiffness: 300, damping: 10))` applied to `NotchOverlayView`'s vertical offset, looping until the phase leaves `waitingForInput`.

### 2.11 Security and Privacy Controls

**Anonymous telemetry:** `AnalyticsClient.track(.session_started)` is the only analytics call in the session lifecycle. The event carries no properties — no `session_id`, `working_dir`, `tool_name`, or any content from the session. This satisfies REQ-SEC-05. The Mixpanel call-site audit at release time (AC-SEC-03, FC-SEC-02) verifies that no session content reaches `MixpanelBridge`.

**Session data in memory only:** `SessionStore` holds session data exclusively in process memory. No session state, chat history excerpt, tool name, or working directory path is written to disk by the session monitoring subsystem. `UserDefaults` stores only non-content settings (sound selection, selected screen, hooks toggle).

**No network transmission of session data:** `module:ipc` uses the Unix domain socket exclusively (REQ-INT-02). Session data does not traverse any network interface.

**App Sandbox disabled:** The session monitoring subsystem accesses `~/.claude/` for JSONL files and the Unix socket. This requires App Sandbox to be disabled (ADR-001). The `com.apple.security.app-sandbox` entitlement is absent from the release entitlements file; CI asserts this before signing. Hardened runtime is enabled (`ENABLE_HARDENED_RUNTIME = YES`, ADR-002); the session monitoring code uses no APIs that conflict with hardened runtime.

### 2.12 Non-Functional Requirements

| Requirement | Threshold | Failure criterion |
|---|---|---|
| Socket-to-`@Published` latency | ≤ 100 ms (±10 ms tolerance) | FC-PERF-01 |
| New session detection after process launch | ≤ 2 s | AC-MON-03 |
| `/clear` command session removal | ≤ 100 ms after `session_end` event | AC-MON-05 |
| Concurrent sessions supported | ≥ 3 | AC-MON-01 |
| Invalid transition handling | No crash; `os_log` error emitted; phase unchanged | AC-MON-02 |
| Missing/corrupt JSONL file | Empty/error state, no crash | FC-REL-02 |
| Main thread blocking by JSONL parse | Must not block | FC-PERF-03 |
| Overlay animation frame rate | ≥ 60 fps | FC-PERF-02 |

---

## 3. Open Questions

**OQ-SLC-01 — Orphaned Session PID Detection Mechanism**
The 30-second reaper that checks for orphaned sessions (§2.5) requires a mechanism to map `Session.id` back to an OS process ID. The hook script does not currently include `$PPID` or `$$` in the `session_start` event JSON. Without a PID, the reaper cannot query `kill -0 <pid>` to confirm liveness. Options include: adding a `pid` field to the `session_start` event (requires hook script update and a new `CLAUDE_ISLAND_HOOK_VERSION`), relying solely on `session_end` events (risking permanently stale sessions if the process is killed without running the hook), or using `NSWorkspace` process enumeration with heuristic matching by `working_dir`. The chosen approach must be reflected in the hook script specification and the session model before the monitoring subsystem ships.

**OQ-SLC-02 — Sequential vs. Concurrent Permission Requests Within a Single Session**
The current model stores a single `pendingPermission: PermissionRequest?` per session. If Claude Code emits a second `tool_call` event for the same session while the first is still `waitingForInput` (which can occur if the session runs multiple tools in a concurrent tool-use block), the second request cannot be stored without overwriting the first. The event processing pipeline (§2.4) logs an `os_log` warning and enqueues the second request, but the queue data structure and draining logic are not yet fully specified. The `PermissionRequestView` must also be extended to indicate "1 of N pending requests." This must be resolved before release if Claude Code can issue concurrent tool calls within a single session.

**OQ-SLC-03 — tmux Pane Routing Protocol Finalization (cross-reference OQ-007 in system design, OQ-IPC-03 in IPC design)**
Section 2.6 specifies that routing correctness is guaranteed by writing the response to the originating file descriptor. However, if the hook script process is killed (e.g., by `SIGKILL`) between sending the `tool_call` event and receiving the response, the descriptor is closed on the client side; the response write will fail with `EPIPE`. `PermissionRouter` must handle `EPIPE` gracefully — removing the registry entry, transitioning the session to `processing`, and notifying the UI — without crashing. The exact behavior on `EPIPE` during a response write, and whether the session should be considered ended or returned to `processing`, must be specified and covered by an integration test.

**OQ-SLC-04 — Session Title Derivation for Non-Directory Working Paths**
`Session.title` is derived as `workingDir.lastPathComponent`. For sessions launched at the root (`/`) or home directory (`~`), this produces `/` or the username, which is not useful as a display title. A fallback title strategy — for example, using the Claude Code process name, the tmux session name if available, or a sequentially numbered "Session N" label — must be decided before the session list view ships.

**OQ-SLC-05 — Elapsed Time Persistence Across App Restarts**
If Claude Island is quit and relaunched while Claude Code sessions are active, `SessionStore` is empty on relaunch. Hook scripts from in-flight sessions will send their next event (e.g., `tool_call`) to the freshly bound socket, creating a new `Session` with `elapsedTime = 0`. The elapsed time from before the restart is lost. For the monitoring use case this is acceptable, but the overlay will show an incorrect (too short) elapsed time for sessions that were running before the restart. Whether this is acceptable or whether `elapsedTime` should be persisted (e.g., in a process-keyed `UserDefaults` entry) is undecided and must be evaluated against user expectations before 1.0 GA.
