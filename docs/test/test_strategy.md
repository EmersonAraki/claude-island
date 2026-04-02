---
codd:
  node_id: test:test-strategy
  type: test
  depends_on:
  - id: test:acceptance-criteria
    relation: derives_from
    semantic: governance
  - id: detail:session_state_machine
    relation: depends_on
    semantic: technical
  - id: detail:ipc_message_flow
    relation: depends_on
    semantic: technical
  depended_by: []
  conventions:
  - targets:
    - test:test-strategy
    reason: A unit test target must be added to the Xcode project (REQ-TEST-01). Shipping
      without a test target is release-blocking.
  - targets:
    - test:test-strategy
    reason: SessionPhase state machine transitions must be covered by unit tests (REQ-TEST-02)
      and hook/socket communication by integration tests (REQ-TEST-03). Gaps in these
      areas block release.
---

# Test Strategy

## 1. Overview

This document defines the test strategy for Claude Island. It governs the test targets, test types, tooling, coverage scope, and the specific test cases required before any release may ship. Every requirement in `req:claude-island-requirements` must be traceable to a test case. Two invariants are non-negotiable release gates and are enforced by this strategy directly:

- **REQ-TEST-01 / FC-TEST-01**: A `ClaudeIslandTests` unit test target must exist in the Xcode project. Shipping without it is release-blocking.
- **REQ-TEST-02 / FC-TEST-01**: All `SessionPhase` state machine transitions must have passing unit tests in `ClaudeIslandTests`.
- **REQ-TEST-03 / FC-TEST-02**: Hook installation and Unix socket communication logic must be covered by integration tests in `ClaudeIslandIntegrationTests`.

This strategy addresses both the `module:session-monitor` (`SessionStore`, `SessionPhase`) and `module:ipc` (`UnixSocketServer`, `SocketActor`, `MessageFramer`, `PermissionRouter`, `ReconnectCoordinator`, `HookInstaller`) modules, which together constitute the release-critical path for session lifecycle tracking and IPC communication.

### Test Targets

| Xcode Target | Type | Framework | Purpose |
|---|---|---|---|
| `ClaudeIslandTests` | Unit | XCTest | `SessionPhase` transitions, state machine guard, isolated component logic |
| `ClaudeIslandIntegrationTests` | Integration | XCTest | Real `UnixSocketServer`, real `SessionStore`, end-to-end socket paths |
| `ClaudeIslandUITests` | UI / Performance | XCTest, Instruments | SwiftUI rendering, animation frame rate, latency, multi-monitor layout |

The `ClaudeIslandTests` target (REQ-TEST-01) must be present and linked against the main app module. Its absence is FC-TEST-01 and blocks release.

### Tooling

- **XCTest** — primary test framework for all unit and integration tests.
- **XCTestExpectation** — used for asynchronous event assertions (latency, `@Published` updates).
- **XCTNSLogExpectation** — used to assert `os_log` emissions for invalid state-machine transitions.
- **Combine `sink`** — used in integration tests to observe `@Published` property updates on `SessionStore.sessions`.
- **Instruments Time Profiler** — referenced in UI performance tests for animation frame-rate verification.

### Isolation Policy

- Unit tests in `ClaudeIslandTests` run against the real `SessionStore` with injected `IPCEvent` values. No mock `SessionStore` is permitted.
- Integration tests in `ClaudeIslandIntegrationTests` run against a real `UnixSocketServer` bound to a temporary path under `/tmp/`. No mocked socket layer is permitted for any integration test path.
- `ChatHistoryParser` runs on `ChatActor`. Integration tests that exercise JSONL parsing must not call parser methods on the main thread; doing so would constitute a violation of FC-PERF-03.

---

## 2. Acceptance Criteria

### 2.1 Release Gate: Unit Test Target Existence (REQ-TEST-01)

**TC-TARGET-01** — The Xcode project file contains a test target named `ClaudeIslandTests` of type `com.apple.product-type.bundle.unit-test`. The target must be linkable and runnable via `xcodebuild test -scheme ClaudeIsland -destination 'platform=macOS'` with exit code 0. Absence of this target is FC-TEST-01 and blocks release unconditionally.

### 2.2 SessionPhase State Machine Unit Tests (REQ-TEST-02)

All eight of the following test methods must be present in `ClaudeIslandTests` and must pass. Absence of any one of the first six is FC-TEST-01 and blocks release.

Each test constructs a `SessionStore` with a pre-seeded `Session` at the required starting phase, injects the triggering `IPCEvent` via `sessionStore.apply(event:)` called on the main actor, and asserts the resulting phase via `XCTAssertEqual(store.sessions[uuid]?.phase, expectedPhase)`.

| Test method | Transition exercised | Release gate |
|---|---|---|
| `testIdleToProcessingOnToolCall` | `idle → processing` on `tool_call` event | FC-TEST-01 |
| `testProcessingToWaitingOnPermissionRequest` | `processing → waitingForInput` on `tool_call` event | FC-TEST-01 |
| `testWaitingToProcessingOnApproval` | `waitingForInput → processing` on `PermissionRouter.respond` with `.allow` | FC-TEST-01 |
| `testProcessingToEndedOnSessionEnd` | `processing → ended` on `session_end` event | FC-TEST-01 |
| `testWaitingToEndedOnSessionEnd` | `waitingForInput → ended` on `session_end` event | FC-TEST-01 |
| `testIdleToEndedOnImmediateSessionEnd` | `idle → ended` on `session_end` event | FC-TEST-01 |
| `testInvalidTransitionEndedToProcessingRejected` | `ended → processing` rejected; phase unchanged; `os_log` error emitted at subsystem `com.claudeisland.session`, category `state-machine` | AC-MON-02 |
| `testInvalidTransitionEndedToWaitingRejected` | `ended → waitingForInput` rejected; phase unchanged; `os_log` error emitted | AC-MON-02 |

**TC-SM-09 — `testClearCommandSessionRemoval`**: inject a `session_end` event directly into `SessionStore` for a session seeded at `.processing`. Assert `sessions[uuid]?.phase == .ended` within 100 ms measured by `XCTestExpectation` with a `Combine` sink `fulfill` on the `@Published` update. This covers REQ-MON-05 / AC-MON-05.

**TC-SM-10 — `testApplyTransitionAllowList`**: construct a `SessionStore` and programmatically exercise every combination of `(currentPhase, targetPhase)` for the six allowed and six explicitly blocked transitions. Assert allowed transitions produce the correct phase and blocked transitions leave the phase unchanged with no crash. This exhaustively validates the `switch (currentPhase, targetPhase)` allow-list in `applyTransition(_:to:)`.

### 2.3 IPC and Hook Integration Tests (REQ-TEST-03)

All twelve of the following test cases must be present in `ClaudeIslandIntegrationTests` and must pass. Absence of any single case is FC-TEST-02 and blocks release. All tests operate against a real `UnixSocketServer` bound to a temporary `/tmp/` path; no mocked socket layer is used.

| Test case | What it verifies |
|---|---|
| `testSocketConnect` | `UnixSocketServer` binds; test client `connect()` succeeds |
| `testSessionStartEventParsed` | `session_start` frame (`{"type":"session_start","session_id":"<uuid>","working_dir":"/path","tmux_pane":"%3"}`) received; `SessionStore.sessions[uuid]?.phase == .idle` within 100 ms |
| `testToolCallEventHoldsConnection` | `tool_call` frame received; connection file descriptor remains open; `SessionStore.sessions[uuid]?.phase == .waitingForInput` |
| `testToolCallResponseDelivered` | Simulated user approval via `PermissionRouter.respond(requestId:decision:.allow)` causes `{"request_id":"...","decision":"allow","response_text":null}` JSON delivery to the open connection file descriptor; connection then closed |
| `testSessionEndEventClosesSession` | `session_end` frame received; `sessions[uuid]?.phase == .ended` |
| `testDisconnectAndReconnect` | Socket file at `/tmp/test-claude-island.sock` deleted; `ReconnectCoordinator` rebinds within 10 seconds; new test client `connect()` succeeds; `socketState == .listening`; no app restart (REQ-REL-01) |
| `testConsentGateBlocksInstall` | `HookInstaller.install()` called with `UserDefaults["hooksConsentGranted"] == false`; throws `HookInstallerError.consentNotGranted`; no files written to `~/.claude/hooks/` |
| `testHookInstallWritesScripts` | With `UserDefaults["hooksConsentGranted"] == true`, `HookInstaller.install()` creates executable scripts at the correct path (`chmod 0755`); `CLAUDE_ISLAND_HOOK_VERSION` variable present in installed scripts |
| `testHookUninstallRemovesScripts` | `HookInstaller.uninstall()` removes all scripts written by `install()` from `~/.claude/hooks/`; `UserDefaults["hooksEnabled"] == false`; `hooksConsentGranted` preserved |
| `testHookUpgradeOverwritesOldVersion` | Install v1 hook scripts; call `HookInstaller.upgrade()` with v2 bundle; assert installed scripts contain v2 `CLAUDE_ISLAND_HOOK_VERSION` string |
| `testConcurrentTwoPaneSessions` | Two test clients connect simultaneously, each sending a `tool_call` frame with distinct `request_id` and `tmux_pane` values (`%3` and `%5`); each client receives only the response frame matching its own `request_id`; no cross-delivery (FC-PERM-01) |
| `testLatencySocketToPublished` | Test client writes a `session_start` frame to a real `UnixSocketServer` bound at a temporary `/tmp/` path; wall-clock elapsed time from socket write to `@Published` update on `SessionStore.sessions` via a `Combine` sink is ≤ 110 ms (100 ms budget + 10 ms tolerance); failure is FC-PERF-01 (REQ-PERF-02) |

### 2.4 Performance and UI Tests

**TC-PERF-01 — `testOverlayAnimationFrameRate`**: use XCTest performance measurement on `NotchOverlayView` during phase transitions. Assert minimum 60 fps sustained on Apple Silicon MacBook with notch. Verify using Instruments Time Profiler that no frame budget is exceeded (AC-UI-03, REQ-PERF-01). Failure is FC-PERF-02.

**TC-PERF-02 — `testJSONLParseOffMainThread`**: call `ChatHistoryParser` with a JSONL file containing ≥ 500 messages; assert execution occurs on `ChatActor` and does not block the main thread; assert no SwiftUI frame drop occurs during parse. Failure is FC-PERF-03.

**TC-PERF-03 — `testConcurrentThreeSessionMonitoring`**: connect three simultaneous test hook clients, each sending independent `session_start` → `tool_call` → `tool_result` → `session_end` sequences. Assert all three sessions display correct independent phase, title, and elapsed time throughout (AC-MON-01).

### 2.5 Reliability Tests

**TC-REL-01 — `testSocketReconnectNoRestart`**: exercise the full `ReconnectCoordinator` retry sequence (`testDisconnectAndReconnect` from §2.3 above). Assert exponential back-off schedule: first attempt at 250 ms, second at 500 ms, third at 1 s, fourth at 2 s, fifth at 4 s, sixth at 8 s (capped). Assert total reconnection time ≤ 10 s. Assert `socketState` transitions through `.reconnecting(attempt: N)` to `.listening`. Failure is FC-REL-01.

**TC-REL-02 — `testMissingJSONLFileShowsEmptyState`**: provide `ChatHistoryParser` with a non-existent JSONL path; assert no crash; assert empty `[ChatMessage]` array returned. Failure is FC-REL-02.

**TC-REL-03 — `testCorruptJSONLFileShowsErrorState`**: provide `ChatHistoryParser` with a JSONL file containing malformed JSON on one line; assert no crash; assert partial parse succeeds for valid lines and an error state is emitted for the malformed line. Failure is FC-REL-02.

**TC-REL-04 — `testDisplayDisconnectionNoCrash`**: simulate removal of the active `NSScreen` during runtime; assert no crash; assert overlay falls back to an available screen; assert `SettingsView` screen picker updates. Failure is FC-REL-03.

### 2.6 Security Tests

**TC-SEC-01 — `testSparkleKeysAbsentFromRepository`**: CI step asserts `.sparkle-keys/` is listed in `.gitignore` and absent from current HEAD. Failure is FC-SEC-01.

**TC-SEC-02 — `testHardenedRuntimeEnabled`**: `xcodebuild` output and `.entitlements` file assert `ENABLE_HARDENED_RUNTIME = YES` and `com.apple.security.app-sandbox` absent. Failure is FC-SEC-04.

**TC-SEC-03 — `testAnalyticsPayloadContainsNoSessionContent`**: inspect all `AnalyticsClient.track(...)` call sites; assert no `session_id`, `working_dir`, `tool_name`, or tool input properties appear in any Mixpanel event payload. Only `app-lifecycle` events (`launch`, `session_started`) are permitted. Failure is FC-SEC-02.

**TC-SEC-04 — `testSocketPermissions0600`**: after `UnixSocketServer.bind()`, call `stat()` on `~/.claude/claude-island.sock`; assert file permissions are `0600`. Failure violates the `module:ipc` security control.

**TC-SEC-05 — `testNoNetworkSocketImports`**: CI lint step scans the `module:ipc` source tree and asserts absence of `NWConnection`, `NWListener`, `URLSession`, `CFSocketCreate`, `socket(AF_INET`, `socket(AF_INET6` symbols. Failure is a REQ-INT-02 violation and blocks release.

**TC-SEC-06 — `testSparkleEdDSASignatureVerification`**: provide Sparkle with a delta update signed with an invalid EdDSA key; assert the update is rejected and the user is notified. Valid-signature update must install. Failure violates AC-UPD-02.

### 2.7 Build and Release Pipeline Tests

**TC-BUILD-01**: `scripts/build.sh` produces a signed `.app` bundle using the Developer ID distribution certificate with hardened runtime and no build errors. Verified as a CI step.

**TC-RELEASE-01**: DMG is notarized; `spctl --assess --type exec` returns acceptance. `scripts/create-release.sh` creates a GitHub release and updates `appcast.xml`. Failure is FC-SEC-03.

**TC-COMPAT-01**: app launches and operates on macOS 15.6 on both Apple Silicon and Intel; tested on a machine without a notch to confirm floating pill fallback. Failure is FC-COMPAT-01.

---

## 3. Failure Criteria

The following conditions are release-blocking failures under this test strategy. Any single unresolved failure prevents shipment.

**FC-TEST-01** — The `ClaudeIslandTests` unit test target does not exist in the Xcode project, or any of the six primary `SessionPhase` transition tests (`testIdleToProcessingOnToolCall`, `testProcessingToWaitingOnPermissionRequest`, `testWaitingToProcessingOnApproval`, `testProcessingToEndedOnSessionEnd`, `testWaitingToEndedOnSessionEnd`, `testIdleToEndedOnImmediateSessionEnd`) is absent or failing. Violates REQ-TEST-01 and REQ-TEST-02.

**FC-TEST-02** — Any of the twelve integration tests in §2.3 is absent from `ClaudeIslandIntegrationTests` or fails. Violates REQ-TEST-03. Specifically includes `testSocketConnect`, `testSessionStartEventParsed`, `testToolCallEventHoldsConnection`, `testToolCallResponseDelivered`, `testSessionEndEventClosesSession`, `testDisconnectAndReconnect`, `testConsentGateBlocksInstall`, `testHookInstallWritesScripts`, `testHookUninstallRemovesScripts`, `testHookUpgradeOverwritesOldVersion`, `testConcurrentTwoPaneSessions`, and `testLatencySocketToPublished`.

**FC-PERF-01** — `testLatencySocketToPublished` reports elapsed wall-clock time from socket write to `@Published` update exceeding 110 ms (100 ms budget + 10 ms tolerance). Violates REQ-PERF-02 and the `module:ipc` / `module:session-monitor` invariant.

**FC-PERF-02** — `testOverlayAnimationFrameRate` measures frame rate below 60 fps on a supported Apple Silicon MacBook with notch during `SessionPhase` transitions. Violates REQ-PERF-01.

**FC-PERF-03** — `testJSONLParseOffMainThread` detects `ChatHistoryParser` executing on the main thread or causing a SwiftUI frame drop during JSONL parsing. Violates REQ-PERF-03.

**FC-REL-01** — `testSocketReconnectNoRestart` fails: `ReconnectCoordinator` does not rebind within 10 seconds of socket file deletion, or the app requires a restart to restore session monitoring. Violates REQ-REL-01.

**FC-REL-02** — `testMissingJSONLFileShowsEmptyState` or `testCorruptJSONLFileShowsErrorState` produces a crash instead of an empty or error state. Violates REQ-REL-02.

**FC-REL-03** — `testDisplayDisconnectionNoCrash` produces a crash on `NSScreen` removal. Violates REQ-REL-03.

**FC-PERM-01** — `testConcurrentTwoPaneSessions` delivers a response frame to the wrong test client. Violates REQ-PERM-03 and REQ-PERM-05. The `PermissionRouter` `request_id`-keyed descriptor registry must guarantee exact-once delivery to the originating file descriptor.

**FC-HOOK-01** — `testConsentGateBlocksInstall` fails: `HookInstaller.install()` writes files to `~/.claude/hooks/` without `UserDefaults["hooksConsentGranted"] == true`. Violates REQ-HOOK-01.

**FC-SEC-01** — `testSparkleKeysAbsentFromRepository` finds `.sparkle-keys/` content in the repository HEAD or untracked in the working directory. Violates REQ-SEC-04.

**FC-SEC-02** — `testAnalyticsPayloadContainsNoSessionContent` finds `session_id`, `working_dir`, `tool_name`, tool input, file path, or any personally identifiable data in any Mixpanel event payload. Violates REQ-SEC-05.

**FC-SEC-03** — `TC-RELEASE-01` reports `spctl --assess --type exec` rejection for the release binary (notarization failure). Violates REQ-SEC-03 and REQ-REL-PIPE-01.

**FC-SEC-04** — `testHardenedRuntimeEnabled` finds `ENABLE_HARDENED_RUNTIME` absent or `com.apple.security.app-sandbox` present in the release entitlements. Violates REQ-SEC-02.

**FC-COMPAT-01** — `TC-COMPAT-01` finds the app failing to launch or crashing on macOS 15.6 on Apple Silicon, Intel, or on a machine without a notch. Violates REQ-COMPAT-01, REQ-COMPAT-02, and REQ-COMPAT-03.
