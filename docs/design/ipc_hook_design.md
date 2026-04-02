---
codd:
  node_id: design:ipc-hook-design
  type: design
  depends_on:
  - id: design:system-design
    relation: depends_on
    semantic: technical
  depended_by:
  - id: detail:ipc_message_flow
    relation: depends_on
    semantic: technical
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:ipc-hook-design
    reason: All hook-to-app communication must use a local Unix domain socket exclusively;
      no network sockets are permitted (REQ-INT-02). Socket reconnection must be automatic
      and require no app restart (REQ-HOOK-04, REQ-REL-01).
  - targets:
    - design:ipc-hook-design
    reason: Hook installation into ~/.claude/hooks/ must require explicit user consent
      on first launch (REQ-HOOK-01). The hooks installer toggle must support runtime
      enable/disable (REQ-HOOK-03).
  - targets:
    - design:ipc-hook-design
    reason: Hook installation and socket communication logic must be covered by integration
      tests (REQ-TEST-03). Untested IPC paths block release.
---

# IPC and Hook Integration Design

## 1. Overview

Claude Island monitors Claude Code CLI sessions by combining two tightly coupled subsystems: a Unix domain socket server (`module:ipc`) that receives newline-delimited JSON events from running hook scripts, and a `HookInstaller` component that manages the lifecycle of those scripts in `~/.claude/hooks/`. Together they form the single communication channel between the CLI and the SwiftUI overlay.

**Scope of this document:** the wire protocol, socket server, reconnection policy, hook installer consent flow, runtime toggle, and the integration-test requirements that gate release. All other subsystems (session model, permission routing, analytics) are described in `design:system-design`; this document treats them as callers and consumers.

**Release-blocking constraints:**

| ID | Constraint | Enforcement |
|---|---|---|
| REQ-INT-02 | Unix domain socket only; no TCP/UDP or network sockets permitted | `UnixSocketServer` binds exclusively to `~/.claude/claude-island.sock`; CI lint rejects `NWConnection` / `URLSession` imports in `module:ipc` |
| REQ-HOOK-01 / FC-HOOK-01 | No files written to `~/.claude/hooks/` without explicit user consent | `HookInstaller.install()` is unreachable unless the consent gate in `SettingsView` has been confirmed; absence of the consent check is a release-blocking failure |
| REQ-HOOK-03 | Hooks installer toggle supports runtime enable/disable | Toggle state is reflected immediately; disabling removes or deactivates installed scripts without requiring a restart |
| REQ-HOOK-04 / REQ-REL-01 / FC-REL-01 | Socket reconnection is automatic; no app restart required | `ReconnectCoordinator` reconnects within 10 s using exponential back-off; absence of this behavior is a release-blocking failure |
| REQ-TEST-03 / FC-TEST-02 | Hook installation and socket communication covered by integration tests | Integration test target must include: socket connect, event send, event receive, disconnect-and-reconnect; absence blocks release |

---

## 2. Architecture

### 2.1 Socket Transport Layer

All IPC between hook scripts and the app uses a local Unix domain socket at the fixed path `~/.claude/claude-island.sock`. No TCP or UDP socket is created at any point (REQ-INT-02). The app is the server; each hook script invocation is a short-lived client.

**Binding and listening:**

At application launch, `UnixSocketServer` executes the following sequence:

1. Remove any stale socket file at `~/.claude/claude-island.sock` (handles prior crash or unclean shutdown).
2. Call `socket(AF_UNIX, SOCK_STREAM, 0)`.
3. Call `bind()` with a `sockaddr_un` struct pointing to `~/.claude/claude-island.sock`.
4. Call `listen()` with backlog 16.
5. Dispatch an accept loop on `SocketActor` (a dedicated `actor` type, not the main actor).

The socket file is created with permissions `0600` (owner read/write only) to prevent other local users from injecting events. The parent directory `~/.claude/` is expected to exist; `HookInstaller` creates it if absent when hooks are enabled.

**Frame format:**

Messages are newline-delimited JSON (`\n`-terminated UTF-8). Each frame is a single JSON object. There is no length prefix. The socket reader buffers bytes until a `\n` byte is found, then parses the complete frame. Partial frames are held in a per-connection buffer. The maximum supported frame size is 1 MiB; frames exceeding this limit are discarded, an `os_log` error is emitted, and the connection is closed.

**Event types emitted by hook scripts:**

| Event `type` field | Trigger | Required fields |
|---|---|---|
| `session_start` | Claude Code process begins | `session_id`, `working_dir`, `tmux_pane` (optional) |
| `tool_call` | Claude Code invokes a tool | `session_id`, `tool_name`, `tool_input`, `request_id` |
| `tool_result` | Tool execution completes | `session_id`, `request_id`, `exit_code`, `output` |
| `session_end` | Claude Code process exits | `session_id`, `exit_code` |
| `ask_user` | `AskUserQuestion` invoked | `session_id`, `request_id`, `prompt_text` |

**Response format (server → client):**

For `tool_call` and `ask_user` events the server holds the connection open and writes a response frame once the user acts in the overlay:

```json
{"request_id": "<uuid>", "decision": "allow" | "deny", "response_text": "<text or null>"}
```

For all other event types the server closes the client connection after parsing without writing a response.

**Latency requirement:**

End-to-end latency from hook script socket write to `@Published` property update on the main actor must not exceed 100 ms (REQ-PERF-02, AC-MON-PERF, FC-PERF-01). `SocketActor` receives and parses frames on a background actor; it then calls `SessionStore` (main actor) via a Swift `async` task dispatch. No synchronous main-thread blocking is permitted in this path.

### 2.2 Reconnection Policy — ReconnectCoordinator

`ReconnectCoordinator` is responsible for maintaining continuous socket availability. Its behavior covers two scenarios:

**Scenario A — Hook script disconnects (expected):** Hook scripts are short-lived; each invocation opens and closes a connection. These are normal teardowns and require no reconnect action on the server side. The accept loop continues listening.

**Scenario B — Server socket becomes unavailable:** If the socket file is deleted (e.g., by the user or an external process), `UnixSocketServer` detects the failure via a `DispatchSource` file-system monitor on `~/.claude/claude-island.sock`. On detection:

1. `ReconnectCoordinator` cancels the current accept loop.
2. It attempts to re-bind and re-listen using an exponential back-off schedule: 250 ms → 500 ms → 1 s → 2 s → 4 s → 8 s (cap). Subsequent attempts hold at 8 s.
3. Full reconnect (new socket bound, accept loop running) must complete within 10 seconds of the disruption (REQ-REL-01, REQ-HOOK-04, AC-HOOK-04).
4. No user interaction and no app restart is required (FC-REL-01).

`ReconnectCoordinator` exposes a `@Published var socketState: SocketState` (`.listening`, `.reconnecting(attempt:)`, `.failed`) consumed by `SettingsView` to display a status indicator.

### 2.3 Hook Installer — HookInstaller

`HookInstaller` manages the presence of hook scripts in `~/.claude/hooks/`. It is the sole component that reads from or writes to that directory.

**Consent gate (REQ-HOOK-01, FC-HOOK-01):**

`HookInstaller.install()` must not be called unless the user has explicitly confirmed the consent dialog. The call path is:

```
SettingsView (hooks toggle ON)
  → ConsentDialogPresenter.requestConsent()
    → user confirms alert ("Enable Hooks" button)
      → UserDefaults.set(true, forKey: "hooksConsentGranted")
        → HookInstaller.install()
```

If the user dismisses or denies the dialog, `hooksConsentGranted` remains `false`, `HookInstaller.install()` is never called, and no files are written to `~/.claude/hooks/`. The toggle reverts to OFF in the UI. This invariant is enforced by a unit test that verifies `HookInstaller.install()` throws `HookInstallerError.consentNotGranted` when called without the `UserDefaults` flag set.

**Runtime enable/disable (REQ-HOOK-03):**

The toggle in the Settings panel reflects the current enable state in real time. Toggling ON:

1. Shows the consent dialog (first time) or skips it (consent already granted).
2. Calls `HookInstaller.install()` which writes hook scripts to `~/.claude/hooks/`.
3. Persists toggle state `true` in `UserDefaults` key `"hooksEnabled"`.

Toggling OFF:

1. Calls `HookInstaller.uninstall()` which removes or chmod-000s the installed scripts.
2. Persists toggle state `false` in `UserDefaults` key `"hooksEnabled"`.
3. Does not revoke `hooksConsentGranted`; re-enabling does not show the consent dialog again.

Toggle state persists across restarts (AC-HOOK-03). On launch, if `hooksEnabled == true` and installed scripts are absent (e.g., user manually deleted them), `HookInstaller.install()` is called silently to restore them.

**Installed scripts:**

`HookInstaller` bundles hook scripts inside the app bundle at `ClaudeIsland.app/Contents/Resources/hooks/`. At install time it copies them to `~/.claude/hooks/` and sets them executable (`chmod 0755`). The scripts are shell scripts (bash, POSIX-compatible) that:

1. Connect to `~/.claude/claude-island.sock` using `socat` or a fallback pure-bash `/dev/tcp` equivalent (if `socat` is absent).
2. Emit the appropriate JSON event frame followed by `\n`.
3. For `tool_call` and `ask_user` events: wait for a response frame, then exit with the appropriate code or write the response text to stdout for Claude Code to consume.
4. For `session_start`, `tool_result`, `session_end` events: exit 0 immediately after sending.

**Hook script versioning:** Hook scripts carry an embedded `CLAUDE_ISLAND_HOOK_VERSION` variable (e.g., `CLAUDE_ISLAND_HOOK_VERSION=1`). On app launch, `HookInstaller` reads the installed version and the bundled version. If they differ, `HookInstaller.upgrade()` overwrites the installed scripts with the bundled versions. This handles the Sparkle-update case where the app binary and its bundled hook scripts are updated together. (Resolution of OQ-006 is required to finalize migration semantics for pre-existing third-party hook scripts.)

### 2.4 Message Flow — End to End

The following sequence covers the `tool_call` event, which is the most complex path:

```
Claude Code CLI
  │  (executes hook script via claude hooks system)
  ▼
~/.claude/hooks/pre-tool-call.sh
  │  connect() to ~/.claude/claude-island.sock
  │  write JSON: {"type":"tool_call","session_id":"…","tool_name":"Bash","tool_input":"…","request_id":"uuid-1"}
  │  blocking read() waiting for response
  ▼
UnixSocketServer (SocketActor — background)
  │  accept() → new client fd
  │  read bytes → MessageFramer buffers until \n
  │  parse JSON → IPCEvent.toolCall(sessionId, toolName, toolInput, requestId)
  │  dispatch to SessionStore (main actor, async)
  ▼
SessionStore (main actor)
  │  update Session.phase = .waitingForInput
  │  update Session.pendingPermission = PermissionRequest(…)
  │  @Published triggers SwiftUI update
  ▼
PermissionRequestView (SwiftUI, main thread)
  │  displays tool name, tool input to user
  │  user taps Allow or Deny
  ▼
PermissionRouter
  │  resolves requestId → client fd (held in open-connection registry)
  │  writes JSON: {"request_id":"uuid-1","decision":"allow","response_text":null}
  │  closes client fd
  ▼
pre-tool-call.sh
  │  read() returns response JSON
  │  parses "decision": "allow" → exit 0  (allow) or exit 2  (deny)
  ▼
Claude Code CLI
  └  proceeds or aborts tool call based on hook exit code
```

The open-connection registry is a dictionary `[String: FileDescriptor]` keyed by `request_id`, maintained on `SocketActor`. Connections for `tool_call` and `ask_user` events are retained in this registry until a response is written. All other connections are closed immediately after parsing. The registry is bounded: entries older than 120 seconds are expired, the connection is closed with a timeout-deny response, and an `os_log` warning is emitted.

### 2.5 tmux Pane Routing

When Claude Code runs inside tmux, the hook script embeds the current pane identifier (read from `$TMUX_PANE`) in every event frame as `"tmux_pane": "%3"`. `SessionStore` stores this value in `Session.tmuxPane`. `PermissionRouter` uses `tmuxPane` to direct the approval response to the correct pane when multiple Claude Code sessions run simultaneously in separate panes (AC-PERM-05). Misrouting is FC-PERM-01 (release-blocking). The routing logic and the tmux-pane field contract are finalized in OQ-007's resolution; the integration tests in §2.6 must cover the two-pane concurrent case before release.

### 2.6 Integration Test Requirements (REQ-TEST-03, FC-TEST-02)

The integration test target `ClaudeIslandIntegrationTests` must include all of the following test cases. Absence of any one case is FC-TEST-02, a release-blocking failure.

| Test case | What it verifies |
|---|---|
| `testSocketConnect` | `UnixSocketServer` binds successfully; a test client can `connect()` |
| `testSessionStartEventParsed` | Client sends `session_start` JSON; `SessionStore` contains the new session within 100 ms |
| `testToolCallEventHoldsConnection` | Client sends `tool_call`; connection remains open until `PermissionRouter` writes response |
| `testToolCallResponseDelivered` | After simulated user approval, response JSON is received by the test client on the open connection |
| `testSessionEndEventClosesSession` | Client sends `session_end`; session phase transitions to `.ended` |
| `testDisconnectAndReconnect` | Server socket file is deleted; `ReconnectCoordinator` re-binds within 10 s; new test client connects successfully |
| `testConsentGateBlocksInstall` | Calling `HookInstaller.install()` without consent throws `HookInstallerError.consentNotGranted` |
| `testHookInstallWritesScripts` | With consent granted, `HookInstaller.install()` creates executable scripts in `~/.claude/hooks/` |
| `testHookUninstallRemovesScripts` | `HookInstaller.uninstall()` removes all scripts written by `install()` |
| `testHookUpgradeOverwritesOldVersion` | Installing version 1 then calling `upgrade()` with version 2 bundle overwrites scripts; new version string is present |
| `testConcurrentTwoPaneSessions` | Two clients each send `tool_call` with distinct `tmux_pane` values; responses are delivered to the correct client |
| `testLatencySocketToPublished` | Time from socket write to `@Published` update on main actor is ≤ 100 ms (tolerance ±10 ms) |

Tests run against a real `UnixSocketServer` instance bound to a temporary socket path under `/tmp/` to avoid interference with a running app. No mocked socket layer is used for the IPC path; the socket transport must be exercised end-to-end (consistent with the project's integration-test policy against mock divergence from production behavior).

### 2.7 Security Controls

**Socket file permissions:** `~/.claude/claude-island.sock` is created with `0600`. Only the owning user (UID match) can connect. No setuid or world-readable socket file is permitted.

**No network exposure:** `module:ipc` contains no `NWConnection`, `URLSession`, or `CFSocket` (TCP/UDP) usage. CI lint step asserts these symbols are absent from the `module:ipc` source tree (REQ-INT-02).

**Hook script integrity:** Hook scripts are copied from the signed app bundle. They are not downloaded from the network at runtime. No dynamic hook script content is fetched or executed.

**Hardened runtime compatibility:** The Unix socket API (`socket`, `bind`, `listen`, `accept`, `read`, `write`, `close`) is fully compatible with hardened runtime (ADR-002). No entitlement exceptions are required for IPC.

**App Sandbox disabled:** Unix socket access to arbitrary paths in `~/` requires App Sandbox to be disabled (ADR-001). The `com.apple.security.app-sandbox` entitlement is absent from the release entitlements file; CI asserts this before code signing.

**Frame size limit:** Frames exceeding 1 MiB are discarded to prevent unbounded memory growth from a malformed or malicious client. The connection is closed on oversize frame detection.

### 2.8 Non-Functional Requirements

| Requirement | Threshold | Failure criterion |
|---|---|---|
| Socket-to-`@Published` latency | ≤ 100 ms (±10 ms tolerance) | FC-PERF-01 |
| Reconnection after socket unavailability | ≤ 10 s; no restart required | FC-REL-01 |
| Concurrent sessions supported | ≥ 3 simultaneous hook script clients | AC-MON-01 |
| Hook consent gate | Zero files written without explicit user confirmation | FC-HOOK-01 |
| Integration test coverage | All 12 test cases listed in §2.6 must pass | FC-TEST-02 |

---

## 3. Open Questions

**OQ-IPC-01 — socat Dependency for Hook Scripts**
The hook scripts rely on `socat` as the primary mechanism for opening a Unix domain socket from a shell script. `socat` is not installed by default on macOS. The fallback (`/dev/tcp`-style bash redirection) does not support `AF_UNIX` on macOS. The alternatives — bundling a minimal `socat` binary inside the app, requiring users to install it via Homebrew, or rewriting the hook scripts as a compiled Swift/Go helper binary — each have distribution and signing implications. This must be resolved before the hook installer ships. A compiled helper binary is the most robust option and would also enable richer event framing, but requires a new ADR covering helper-binary signing and notarization.

**OQ-IPC-02 — Hook Script Versioning and Migration for Pre-Existing Installations (ref. OQ-006 in system design)**
When a Sparkle update delivers a new app version with updated bundled hook scripts, `HookInstaller.upgrade()` overwrites scripts that Claude Island previously installed. However, the behavior for hook scripts that were not installed by Claude Island (user-written hooks, scripts from other tools) is undefined. The current implementation writes only to `~/.claude/hooks/pre-tool-call.sh`, `pre-bash.sh`, and analogous filenames; collisions with user scripts at those exact names would silently overwrite user work. A naming convention (e.g., `claude-island-pre-tool-call.sh`) or a subdirectory under `~/.claude/hooks/` is needed. Resolution required before the hook installer toggle ships.

**OQ-IPC-03 — tmux Pane Routing Protocol Finalization (ref. OQ-007 in system design)**
The `tmux_pane` field in event JSON is specified as the raw `$TMUX_PANE` value (e.g., `%3`). The `PermissionRouter`'s strategy for delivering the approval response to the correct pane — whether by writing to a named pipe, by calling `tmux send-keys`, or by another mechanism — is not yet specified. The integration test `testConcurrentTwoPaneSessions` (§2.6) requires a concrete implementation to pass. FC-PERM-01 is release-blocking. The routing protocol must be finalized, reflected in the hook script source, and covered by the integration test before the hooks installer toggle ships.

**OQ-IPC-04 — Connection Timeout Behavior for Long-Running Permission Requests**
The 120-second timeout on open `tool_call` and `ask_user` connections (§2.4) results in a deny response being sent to the hook script after expiry. This timeout is appropriate for an unattended overlay, but the user experience for a request that times out is unspecified: the overlay may still be showing the permission dialog when the timeout fires and closes the connection. The UI must be updated to reflect the timeout (e.g., dismiss the dialog, show a "request expired" state). The interaction between `ReconnectCoordinator` and the connection registry during a rebind cycle (where open connections from the previous socket file descriptor become invalid) also needs a defined flush policy.

**OQ-IPC-05 — Socket Path Configurability**
The socket path `~/.claude/claude-island.sock` is hardcoded in both the app and the bundled hook scripts. Users who run multiple Claude Island instances (e.g., for testing a beta build alongside a stable release) would experience socket path conflicts. A `CLAUDE_ISLAND_SOCKET_PATH` environment variable override, read by both the app and the hook scripts, would allow coexistence. This is a low-priority quality-of-life improvement but should be decided before 1.0 GA so that the socket path is not permanently baked into distributed hook scripts that cannot be updated without consent.
