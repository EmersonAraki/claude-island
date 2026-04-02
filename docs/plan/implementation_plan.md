---
codd:
  node_id: plan:implementation-plan
  type: plan
  depends_on:
  - id: test:acceptance-criteria
    relation: constrained_by
    semantic: governance
  - id: test:test-strategy
    relation: depends_on
    semantic: technical
  - id: detail:notch_overlay_state_machine
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
  - id: design:release-pipeline-design
    relation: depends_on
    semantic: technical
  depended_by: []
  conventions:
  - targets:
    - plan:implementation-plan
    reason: Plan must include a dedicated phase for adding the unit test target and
      achieving SessionPhase test coverage (REQ-TEST-01, REQ-TEST-02). Omitting this
      phase blocks release.
  - targets:
    - plan:implementation-plan
    reason: Release phase must include notarization, DMG generation, Sparkle EdDSA
      signing, GitHub release creation, and appcast update steps in order (REQ-REL-PIPE-01
      through REQ-REL-PIPE-05). Shipping without the full pipeline is release-blocking.
  - targets:
    - plan:implementation-plan
    reason: Plan must account for Sparkle private key management outside VCS from
      day one (REQ-SEC-04). Committing keys even temporarily is a release-blocking
      security violation.
---

# Implementation Plan

## 1. Overview

Claude Island is a macOS SwiftUI overlay application that surfaces Claude Code CLI session state in a notch-anchored (or floating-pill) window. This plan sequences the work required to reach a shippable 1.0 GA build that satisfies every acceptance criterion in `test:acceptance-criteria` and passes every release gate enumerated in `test:test-strategy` and `design:release-pipeline-design`.

The implementation is organized into six sequential phases. Each phase ends with a defined review gate; a phase may not begin until its predecessor's gate is cleared. Two constraints are non-negotiable across all phases and are explicitly reflected in every relevant milestone:

- **REQ-TEST-01 / REQ-TEST-02 (FC-TEST-01):** A `ClaudeIslandTests` unit test target must exist and all `SessionPhase` state-machine transitions must have passing tests before any release build is produced. This is a dedicated phase (Phase 3) and blocks all subsequent phases.
- **REQ-REL-PIPE-01 through REQ-REL-PIPE-05 / REQ-SEC-04 (FC-SEC-01, FC-SEC-03):** The release pipeline executes notarization → DMG generation → Sparkle EdDSA signing → GitHub release creation → appcast update in strict order. The Sparkle EdDSA private key is never committed to VCS. Both constraints are enforced from day one; `.sparkle-keys/` is added to `.gitignore` before the first commit and the key is stored exclusively in the local secrets manager and `.sparkle-keys/` on the release engineer's machine.

**Target platform:** macOS 15.6, Universal Binary (arm64 + x86_64). App Sandbox is permanently disabled (`com.apple.security.app-sandbox = false`); hardened runtime is enabled (`ENABLE_HARDENED_RUNTIME = YES`). Distribution is via Developer ID exclusively.

**SPM dependencies declared in `Package.swift`:** `sparkle-project/Sparkle`, `apple/swift-markdown`, `mixpanel-swift`. No CocoaPods, Carthage, or manual xcframework embedding is permitted (REQ-BUILD-01).

**Analytics isolation (REQ-SEC-05 / FC-SEC-02):** `MixpanelBridge` is the sole import site of `mixpanel-swift`. No type in `Chat/`, `Permission/`, or `IPC/` may reference `AnalyticsClient.track` or import `mixpanel-swift`. Only two zero-property events (`app_launched`, `session_started`) are ever transmitted. This constraint is enforced by a Mixpanel call-site audit before every release.

---

## 2. Milestones

### Phase 0 — Project Bootstrap and Security Baseline (Pre-development gate)

**Objective:** Establish the repository, security controls, and toolchain before any feature code is written. Every constraint in this phase is release-blocking if omitted.

**Tasks:**

1. Initialize Xcode project targeting macOS 15.6, `ARCHS = arm64 x86_64`, `ONLY_ACTIVE_ARCH = NO`.
2. Add `.gitignore` entries for `.sparkle-keys/` and `*.key`, `*.pem`, `*.p8`, `private_key`, `sparkle_private*`. This is the first commit. Committing key material even temporarily violates REQ-SEC-04 and is a release-blocking security violation.
3. Generate Sparkle Ed25519 keypair using `generate_keys` (Sparkle CLI tool). Store private key in `.sparkle-keys/` on the release engineer's machine and in the secrets manager. Embed the public key in `Info.plist` under `SUPublicEDKey`. Confirm `git ls-files .sparkle-keys/` returns empty.
4. Configure `ClaudeIsland.entitlements`: `com.apple.security.app-sandbox = false`, no JIT, no library-validation disable. CI assertion: `grep -c "app-sandbox.*true"` returns `0`.
5. Install `scripts/setup-dev.sh` pre-commit hook that blocks staged files matching key patterns.
6. Add `Package.swift` declaring `Sparkle`, `apple/swift-markdown`, and `mixpanel-swift`.
7. Configure GitHub Actions CI with required status checks: `trufflehog` secret scan, entitlements assertion, hardened-runtime flag assertion, Universal Binary verification via `lipo -info`.
8. Add `scripts/build.sh` stub: `xcodebuild archive` with `ENABLE_HARDENED_RUNTIME=YES`, `CODE_SIGN_IDENTITY="Developer ID Application"`, `MACOSX_DEPLOYMENT_TARGET=15.6`.
9. Add `scripts/create-release.sh` stub with all eight pipeline stages wired in sequence: clean build → code sign → DMG creation → notarization → staple → Sparkle EdDSA sign → GitHub release upload → appcast update. Stages 4–8 are no-ops until a real build exists but the gate ordering is enforced from the start.

**Gate:** CI green on an empty-app build. `git ls-files .sparkle-keys/` returns empty. Entitlements assertion passes.

---

### Phase 1 — Core Architecture: IPC Layer and Session State Machine

**Objective:** Implement `module:ipc` (`UnixSocketServer`, `SocketActor`, `MessageFramer`, `ReconnectCoordinator`, `HookInstaller`) and `module:session-monitor` (`SessionStore`, `SessionPhase`, `PermissionRouter`) as production-quality components. This phase establishes the critical data path from hook script to `@Published` UI update.

**Tasks:**

1. **`SessionPhase` enum:** Define `idle`, `processing`, `waitingForInput`, `ended` conforming to `Equatable`. Define as a Swift `enum` in `SessionPhase.swift`.

2. **`Session` struct:** Value type. Fields: `id: UUID`, `phase: SessionPhase`, `title: String`, `workingDir: String`, `tmuxPane: String?`, `pendingPermission: PermissionRequest?`, `promptText: String?`, `elapsedStart: Date?`. No in-place mutation; `SessionStore` replaces dictionary entries.

3. **`IPCEvent` value type:** Cases: `sessionStart(id:workingDir:tmuxPane:)`, `toolCall(sessionId:toolName:toolInput:requestId:tmuxPane:)`, `toolResult(sessionId:requestId:exitCode:output:)`, `askUser(sessionId:requestId:promptText:)`, `sessionEnd(sessionId:exitCode:)`.

4. **`MessageFramer`:** Per-connection buffer accumulator. UTF-8 newline-delimited frames. Maximum inbound frame size: 1,048,576 bytes (1 MiB). Frames exceeding the limit are discarded and the connection is closed. `func consume(bytes: Data) -> [InboundFrame]`.

5. **`UnixSocketServer` / `SocketActor`:** `socket(AF_UNIX, SOCK_STREAM, 0)`, `bind()` to `~/.claude/claude-island.sock`, `listen(backlog: 16)`, `accept()` loop on `SocketActor`. Socket file created with permissions `0600`. One `MessageFramer` per accepted `FileDescriptor`; deallocated on connection close. `JSONDecoder` parse runs synchronously on `SocketActor` before the main-actor dispatch. No `NWConnection`, `NWListener`, `URLSession`, `CFSocketCreate`, `socket(AF_INET`, `socket(AF_INET6` in `module:ipc`; CI lint asserts this on every build (REQ-INT-02). Dispatches `IPCEvent` to `SessionStore` via `Task { @MainActor in store.apply(event:fd:) }`.

6. **`ReconnectCoordinator`:** `DispatchSource` file-system monitor on `~/.claude/claude-island.sock`. On deletion/rename: cancel accept loop, set `socketState = .reconnecting(attempt: 1)`, execute exponential back-off: 250 ms → 500 ms → 1 s → 2 s → 4 s → 8 s (capped). Each attempt calls `UnixSocketServer.rebind()`: remove stale file, `socket(AF_UNIX)`, `bind()`, `listen(backlog: 16)`. Full rebind must complete within 10 seconds; after 10 s without success, set `socketState = .failed`. On success: `socketState = .listening`. Open connections active at disruption have their descriptors invalidated; `PermissionRouter` detects `EPIPE`, removes the registry entry, transitions the affected session from `waitingForInput` to `processing`, and emits `os_log` warning.

7. **`PermissionRouter`:** Owned by `SessionStore`. `openConnections: [String: FileDescriptor]` keyed by `request_id`. `func register(requestId: String, fd: FileDescriptor)`. `func respond(requestId: String, decision: Decision, responseText: String?)`: writes `{"request_id":"...","decision":"allow|deny","response_text":null}\n` to the originating descriptor; closes descriptor; removes from registry. Connections older than 120 seconds are expired by a 10-second reaper `Timer`: write deny response, close descriptor, remove from registry, transition session to `processing`, display "Request expired" banner for 2 seconds. `PermissionRouter` is not an independent singleton; call sites reach it only through `SessionStore.respondToPermission(requestId:decision:)`.

8. **`HookInstaller`:** Sole component that reads from or writes to `~/.claude/hooks/`. Scripts sourced exclusively from `ClaudeIsland.app/Contents/Resources/hooks/`. `func install() throws`: checks `UserDefaults["hooksConsentGranted"] == true`; throws `HookInstallerError.consentNotGranted` if false; copies scripts with `chmod 0755`; sets `UserDefaults["hooksEnabled"] = true`. `func uninstall()`: removes all installed scripts; sets `UserDefaults["hooksEnabled"] = false`; preserves `hooksConsentGranted`. `func upgrade()`: overwrites installed scripts with bundled version if `CLAUDE_ISLAND_HOOK_VERSION` differs. Hook script naming convention to avoid user-file collisions must be resolved before hook installer ships (OQ-IPC-02); integration tests must assert the chosen path scheme.

9. **`SessionStore`:** `@MainActor final class` conforming to `ObservableObject`. `@Published var sessions: [UUID: Session] = [:]`. `private var timers: [UUID: Timer] = [:]`. `func apply(event: IPCEvent, fd: FileDescriptor?)`: calls `applyTransition(_:to:)` for all phase-changing events. `func applyTransition(_ session: Session, to targetPhase: SessionPhase)`: exhaustive `switch (session.phase, targetPhase)` allow-list for six valid transitions (`idle→processing`, `idle→ended`, `processing→waitingForInput`, `processing→ended`, `waitingForInput→processing`, `waitingForInput→ended`); all other combinations emit `os_log(.error, subsystem: "com.claudeisland.session", category: "state-machine")` and return without mutation. No crash on invalid transitions (AC-MON-02). Timer management: created on `idle→processing` transition; invalidated on `ended` transition; left running on `waitingForInput↔processing`. Session removed from `sessions` 3 seconds after `ended` transition. `AnalyticsClient.track(.session_started)` called on `session_start` event with zero content properties.

10. **`AnalyticsClient` protocol and `MixpanelBridge`:** `MixpanelBridge` is the only file that imports `mixpanel-swift`. Two permitted events: `app_launched` (no properties), `session_started` (no properties). No `session_id`, `working_dir`, `tool_name`, or tool input in any event payload (REQ-SEC-05).

**Latency budget enforcement:** Total socket write to `@Published` update ≤ 100 ms (±10 ms tolerance). Budget allocation: `SocketActor` read + `MessageFramer` assembly ≤ 20 ms; `JSONDecoder` parse on `SocketActor` ≤ 10 ms; `Task @MainActor` dispatch ≤ 30 ms; `SessionStore.apply()` + dictionary replace ≤ 10 ms; SwiftUI diff + render ≤ 30 ms (REQ-PERF-02, FC-PERF-01).

**Gate:** All `module:ipc` and `module:session-monitor` components compile. CI lint confirms no network socket imports in `module:ipc`. Manual smoke test: hook script writes `session_start` frame; `SessionStore` reflects new session.

---

### Phase 2 — Overlay UI and Settings

**Objective:** Implement the notch overlay window, all `OverlayState` transitions, permission request UI, chat history rendering, multi-monitor support, sound notifications, and settings panel.

**Tasks:**

1. **`OverlayState` enum in `NotchOverlayStateMachine.swift`:** Cases: `closed`, `popping`, `opened`, `closing`, `opened_permission`, `popping_permission`, `opened_waitingForInput`, `opened_success`, `opened_error`. No other module redefines or shadows this type.

2. **`NotchOverlayWindow` (`NSWindow`):** `canBecomeKey = false`. `NSApp.activate` never called. Frame calculation: `NSScreen.safeAreaInsets.top > 0` → notch mode (pill width matches notch bounding rect, extends 4 pt below notch bottom edge); otherwise → floating-pill fallback (centered horizontally at top of selected screen, 8 pt top margin, REQ-UI-04, REQ-COMPAT-03). Updates frame on `NSApplication.didChangeScreenParametersNotification`. Absence of floating-pill fallback path is FC-UI-01, a release-blocking failure.

3. **`SessionStore` overlay state API (sole legal mutators):** `openOverlay()` → `closed → popping`; `closeOverlay()` → `opened → closing`; `handleIncomingPermission(_:)` → conditional `→ popping_permission` or `→ opened_permission` (REQ-PERM-04); `acceptPermission(sessionID:)` → `opened_permission → opened`; `denyPermission(sessionID:)` → `opened_permission → opened`; `handlePhaseChange(_:sessionID:)` → content-axis transitions; `handleSuccessTimer()` → `opened_success → closing` (3-second `Timer`).

4. **`SessionStore.handleIncomingPermission(_:)`:** Auto-expansion guard: check `NSWorkspace.shared.frontmostApplication?.bundleIdentifier == session.terminalBundleIdentifier`. If terminal not visible and overlay is `closed`: `overlayState = .poppingPermission`. This implements AC-PERM-04; omitting it is a release-blocking defect.

5. **Spring animations (REQ-PERF-01, REQ-UI-03):** All transitions use `withAnimation(.spring(response: 0.35, dampingFraction: 0.72))`. Blur radius animated separately on `PillShapeBackground` via `.animation(.spring, value: overlayState)` to decouple from geometry layout passes. `matchedGeometryEffect` is prohibited inside `ExpandedContent` (triggers extra layout pass, violates 16.67 ms frame budget). Fixed-frame layout with `frame(width:height:)` mandatory for `PermissionRequestView`. All six transition paths verified by `xcrun xctrace` Time Profiler on Release build (CI step `overlay-perf-gate`); a frame exceeding 16.67 ms is FC-PERF-02 and blocks release.

6. **`ExpandedContent`:** Single SwiftUI view switching between content variants via `@ViewBuilder` keyed on `overlayState`: chat history (default), permission request amber overlay, `waitingForInput` input bar + bounce, `opened_success` green tint 3-second hold, `opened_error` red tint. No separate `NavigationStack` or sheet presentations; all variants are in-place layout swaps within the same `ZStack`.

7. **Crab icon animation:** Animating during `processing` and `waitingForInput` phases; static during `idle` and `ended` (AC-UI-05).

8. **Amber indicator:** Displayed during `waitingForInput` when a permission payload is pending (AC-UI-06).

9. **Green checkmark:** Displayed during `ended` with success result for 3 seconds; not shown on error termination (AC-UI-07).

10. **Overlay bounce:** Visual Y-offset pulse animation `.spring(response: 0.3, dampingFraction: 0.4).repeatCount(1)` when user input is required and terminal window is not in focus (AC-UI-08). No bounce when terminal is focused (AC-MON-06).

11. **`PermissionRequestView`:** Displays tool name and complete tool input within 100 ms of socket message receipt (AC-PERM-01). Approve / Deny buttons call `sessionStore.acceptPermission(sessionID:)` / `sessionStore.denyPermission(sessionID:)`. Response delivered to the correct Claude Code process only (AC-PERM-03). When `session.tmuxPane == nil` at approval time for a session that requires tmux routing, surface a visible warning and block Approve/Deny buttons until routing state is resolved (OQ-SM-002 handling; FC-PERM-01 is release-blocking).

12. **`ChatHistoryParser` on `ChatActor`:** JSONL file I/O and `JSONDecoder` parsing confined to `ChatActor`. An `assert(Thread.isMainThread == false)` fires in debug builds. Results published to main actor via `@MainActor func updateHistory(_ messages: [ChatMessage])`. Missing JSONL file: return `ChatHistoryResult.empty` without crashing (FC-REL-02). Malformed line: partial parse of valid lines; emit error state for malformed line without crashing (FC-REL-02). `SessionStore` cache: `chatHistory: [UUID: [ChatMessage]]` keyed on `UUID` and JSONL file modification date; switching `activeSessionID` reads from cache synchronously without re-parse.

13. **`ChatHistoryView`:** Uses `apple/swift-markdown` (sole import site) for `AssistantMessageView` Markdown rendering: bold, inline code, fenced code blocks, unordered and ordered lists (AC-CHAT-02). Tool calls shown inline with name, arguments, and completion status (AC-CHAT-04). Auto-scrolls to latest message; manual scroll up pauses auto-scroll; scroll-to-bottom affordance resumes it (AC-CHAT-03). Switching sessions and returning shows cached history immediately without re-parsing; no render flicker (AC-CHAT-05). `waitingForInput` (AskUserQuestion): interactive prompt bar displayed; submitting relays text to Claude Code process (AC-CHAT-06). `idle`: message input bar available (AC-CHAT-07).

14. **`AskInputBar`:** Surfaces AskUserQuestion prompt text (AC-ASK-01). User types and submits response without switching to terminal (AC-ASK-02). `canBecomeKey` / `@FocusState` interaction on macOS 15.6 must be validated on physical hardware before AskUserQuestion ships (OQ-SM-001).

15. **`ScreenSelector`:** Wraps `NSScreen.screens`. Persists selected display via `UserDefaults["selectedDisplayIdentifier"]` (CGDirectDisplayID). Fallback chain on disconnect: (1) match by ID; (2) `NSScreen.screens.first`; (3) `NSScreen.main`. Never sets `selectedScreen` to nil; absence of fallback is FC-REL-03. Re-evaluates on `NSApplication.didChangeScreenParametersNotification` (AC-SCREEN-04).

16. **`SoundPlayer`:** Stateless struct. `play(for:terminalHasFocus:)`. Plays user-selected system sound from `/System/Library/Sounds/` on `ended` success when terminal is not frontmost (AC-SOUND-05). No AVFoundation import outside `SoundPlayer`.

17. **`SettingsView`:** Exactly six items (AC-SET-02): screen picker; inline scrollable sound picker listing ≥ 14 system sounds (AC-SOUND-02, AC-SOUND-03); launch-at-login toggle via `SMAppService` (AC-SET-03); hooks installer toggle with consent alert gate (consent must fire `ConsentDialogPresenter.requestConsent` before any `HookInstaller.install()` call — FC-HOOK-01); accessibility permissions status with direct path to System Settings > Privacy & Security > Accessibility (AC-SET-04); auto-update control with green dot indicator for unseen updates (AC-UPD-03). All settings persist across app restarts (AC-SOUND-04, AC-SCREEN-03, AC-SET-03).

18. **`SparkleUpdater`:** Sole import site of `Sparkle` SPM package. Wraps `SPUStandardUpdaterController`. Feed URL: `https://claudeisland.com/appcast.xml`. EdDSA signature verification: update with invalid or missing signature rejected; user notified (AC-UPD-02). `SUPublicEDKey` embedded in `Info.plist`. `updateIsAvailable: Bool` publisher consumed by `SettingsView` gear-button badge (AC-UPD-03). Sparkle private key absent from repository; `.sparkle-keys/` in `.gitignore` (AC-UPD-04, REQ-SEC-04, FC-SEC-01).

**Gate:** All overlay states reachable. Floating-pill fallback verified on non-notch target. Spring animations pass Instruments Time Profiler review. Settings panel shows all six items. Sparkle integration connects to appcast feed.

---

### Phase 3 — Test Targets and Coverage (Release Gate)

**Objective:** Add `ClaudeIslandTests` (unit) and `ClaudeIslandIntegrationTests` (integration) Xcode targets and achieve the full test coverage required by REQ-TEST-01, REQ-TEST-02, and REQ-TEST-03. This phase is a hard release gate; no Phase 4, 5, or 6 work may ship without these tests passing.

**Compliance with REQ-TEST-01 (FC-TEST-01):** The `ClaudeIslandTests` unit test target of type `com.apple.product-type.bundle.unit-test` must exist in the Xcode project and run via `xcodebuild test -scheme ClaudeIsland -destination 'platform=macOS'` with exit code 0. Absence of this target is FC-TEST-01 and blocks release unconditionally.

**`ClaudeIslandTests` — required unit test methods (REQ-TEST-02 / FC-TEST-01):**

All tests construct a `SessionStore` with a pre-seeded `Session`, inject the triggering `IPCEvent` via `sessionStore.apply(event:)` on the main actor, and assert the resulting phase via `XCTAssertEqual`.

| Test method | Transition | Release gate |
|---|---|---|
| `testIdleToProcessingOnToolCall` | `idle → processing` | FC-TEST-01 |
| `testProcessingToWaitingOnPermissionRequest` | `processing → waitingForInput` | FC-TEST-01 |
| `testWaitingToProcessingOnApproval` | `waitingForInput → processing` | FC-TEST-01 |
| `testProcessingToEndedOnSessionEnd` | `processing → ended` | FC-TEST-01 |
| `testWaitingToEndedOnSessionEnd` | `waitingForInput → ended` | FC-TEST-01 |
| `testIdleToEndedOnImmediateSessionEnd` | `idle → ended` | FC-TEST-01 |
| `testInvalidTransitionEndedToProcessingRejected` | `ended → processing` rejected; phase unchanged; `os_log` error at `com.claudeisland.session` / `state-machine` | AC-MON-02 |
| `testInvalidTransitionEndedToWaitingRejected` | `ended → waitingForInput` rejected; phase unchanged; `os_log` error emitted | AC-MON-02 |
| `testClearCommandSessionRemoval` | `session_end` → `.ended` within 100 ms; asserted via `XCTestExpectation` + `Combine` sink | AC-MON-05 |
| `testApplyTransitionAllowList` | Exhaustive `(currentPhase, targetPhase)` allow-list: all six allowed and six explicitly blocked combinations | AC-MON-02 |

No mock `SessionStore` is permitted in unit tests; tests run against the real implementation.

**`ClaudeIslandIntegrationTests` — required integration test cases (REQ-TEST-03 / FC-TEST-02):**

All tests run against a real `UnixSocketServer` bound to a temporary path under `/tmp/`. No mocked socket layer is used.

| Test case | What it verifies |
|---|---|
| `testSocketConnect` | `UnixSocketServer` binds; test client `connect()` succeeds |
| `testSessionStartEventParsed` | `session_start` frame received; `sessions[uuid]?.phase == .idle` within 100 ms |
| `testToolCallEventHoldsConnection` | `tool_call` received; connection fd remains open; `phase == .waitingForInput` |
| `testToolCallResponseDelivered` | `PermissionRouter.respond(requestId:decision:.allow)` causes correct JSON delivery to originating fd |
| `testSessionEndEventClosesSession` | `session_end` received; `phase == .ended` |
| `testDisconnectAndReconnect` | Socket file deleted; `ReconnectCoordinator` rebinds within 10 s; new client connects; `socketState == .listening` |
| `testConsentGateBlocksInstall` | `HookInstaller.install()` with `hooksConsentGranted == false` throws `HookInstallerError.consentNotGranted`; no files written |
| `testHookInstallWritesScripts` | With consent, `install()` creates `chmod 0755` scripts at correct path; `CLAUDE_ISLAND_HOOK_VERSION` present |
| `testHookUninstallRemovesScripts` | `uninstall()` removes all installed scripts; `hooksEnabled == false`; `hooksConsentGranted` preserved |
| `testHookUpgradeOverwritesOldVersion` | Install v1; `upgrade()` with v2 bundle; installed scripts contain v2 `CLAUDE_ISLAND_HOOK_VERSION` |
| `testConcurrentTwoPaneSessions` | Two clients send `tool_call` with distinct `request_id` and `tmux_pane` (`%3`, `%5`); each receives only its own response (FC-PERM-01) |
| `testLatencySocketToPublished` | Wall-clock from socket write to `@Published` update ≤ 110 ms (100 ms + 10 ms tolerance); failure is FC-PERF-01 |

**Additional required tests:**

- `TC-PERF-01 testOverlayAnimationFrameRate`: XCTest performance measurement on `NotchOverlayView` during phase transitions; assert ≥ 60 fps on Apple Silicon with notch; Instruments Time Profiler verification (FC-PERF-02).
- `TC-PERF-02 testJSONLParseOffMainThread`: `ChatHistoryParser` with ≥ 500 messages; assert execution on `ChatActor`; no main-thread block; no SwiftUI frame drop (FC-PERF-03).
- `TC-PERF-03 testConcurrentThreeSessionMonitoring`: Three simultaneous hook clients; all three sessions show correct independent phase, title, and elapsed time (AC-MON-01).
- `TC-REL-01 testSocketReconnectNoRestart`: Full `ReconnectCoordinator` retry sequence; exponential back-off schedule 250 ms → 500 ms → 1 s → 2 s → 4 s → 8 s capped; total ≤ 10 s; `socketState` transitions through `.reconnecting(attempt: N)` to `.listening` (FC-REL-01).
- `TC-REL-02 testMissingJSONLFileShowsEmptyState`: Non-existent JSONL path; no crash; empty `[ChatMessage]` returned (FC-REL-02).
- `TC-REL-03 testCorruptJSONLFileShowsErrorState`: Malformed JSON line; no crash; valid lines parsed; error state emitted for malformed line (FC-REL-02).
- `TC-REL-04 testDisplayDisconnectionNoCrash`: Simulate `NSScreen` removal; no crash; overlay falls back to available screen (FC-REL-03).
- `TC-SEC-01 testSparkleKeysAbsentFromRepository`: CI asserts `.sparkle-keys/` in `.gitignore` and absent from HEAD (FC-SEC-01).
- `TC-SEC-02 testHardenedRuntimeEnabled`: `ENABLE_HARDENED_RUNTIME = YES`; `app-sandbox` absent in entitlements (FC-SEC-04).
- `TC-SEC-03 testAnalyticsPayloadContainsNoSessionContent`: No `session_id`, `working_dir`, `tool_name`, or tool input in any Mixpanel event payload (FC-SEC-02).
- `TC-SEC-04 testSocketPermissions0600`: `stat()` on `~/.claude/claude-island.sock` confirms permissions `0600`.
- `TC-SEC-05 testNoNetworkSocketImports`: CI lint asserts absence of `NWConnection`, `NWListener`, `URLSession`, `CFSocketCreate`, `socket(AF_INET`, `socket(AF_INET6` in `module:ipc` source tree (REQ-INT-02).
- `TC-SEC-06 testSparkleEdDSASignatureVerification`: Invalid EdDSA key update rejected; valid-signature update installs (AC-UPD-02).
- `TC-BUILD-01`: `scripts/build.sh` produces signed `.app` with Developer ID, hardened runtime, no build errors.
- `TC-COMPAT-01`: App launches on macOS 15.6 on Apple Silicon, Intel, and a machine without a notch; floating-pill fallback confirmed (FC-COMPAT-01).

**Gate:** `xcodebuild test` exits 0 for both `ClaudeIslandTests` and `ClaudeIslandIntegrationTests`. All FC-TEST-01 and FC-TEST-02 test cases present and passing. CI green on all required status checks.

---

### Phase 4 — Integration, Hardware Validation, and Bug Fixes

**Objective:** End-to-end validation on physical hardware, resolution of open questions that block feature shipping, and elimination of all release-blocking failure criteria.

**Tasks:**

1. **Hook script connectivity resolution (OQ-IPC-01):** Implement signed compiled Swift helper binary (`ClaudeIsland Helper`) bundled in `ClaudeIsland.app/Contents/Resources/hooks/`. New ADR covering helper-binary signing, notarization, and distribution. Hook installer toggle must not ship to users until this is resolved.

2. **Hook script naming convention (OQ-IPC-02):** Decide between `claude-island-<event-type>.sh` prefixed filenames or `~/.claude/hooks/claude-island/` subdirectory. Update `HookInstaller` to use chosen scheme. Update `testHookInstallWritesScripts` and `testHookUninstallRemovesScripts` to assert correct paths.

3. **`canBecomeKey` / `@FocusState` validation (OQ-SM-001):** Test `canBecomeKey = false` + SwiftUI `@FocusState` on macOS 15.6 physical hardware. If key focus must be temporarily elevated for `AskInputBar`, add sub-state `opened_waitingForInput_focused` that sets `canBecomeKey = true` on entry and resets on exit. Document decision and write covering test before `AskUserQuestion` ships.

4. **EPIPE handling in `PermissionRouter` (OQ-SSM-03 / OQ-IPC-03):** Specify and implement correct session phase after `EPIPE` on response write. Cover by integration test before hooks toggle ships.

5. **`SIGKILL` orphan reaper (OQ-SSM-01):** Implement 30-second orphan reaper in `SessionStore` that maps `session_id` to PID; transitions orphaned sessions to `.ended` when PID is no longer running. AC-MON-05 compliance applies only to clean `/clear` path until resolved.

6. **Notch width validation on hardware (OQ-SM-005):** Validate `NSScreen.safeAreaInsets.top`-derived pill alignment on 14-inch and 16-inch MacBook Pro. Add hardware-specific correction table to `NotchOverlayWindow.updateFrame(for:)` if pixel alignment differs between models.

7. **Connection timeout UI coordination (OQ-IPC-04):** Implement flush policy on `ReconnectCoordinator` rebind: expire all `openConnections` entries immediately with deny, transition affected sessions to `processing`, dismiss any in-progress permission overlay, before new accept loop begins.

8. **`testLatencySocketToPublished` CI enforcement:** Wire the integration test into the CI `overlay-perf-gate` step. Confirm ≤ 110 ms on both Apple Silicon and Intel.

9. **Sound picker and multi-monitor hardware test:** Validate all 14+ system sounds playable from `/System/Library/Sounds/`. Validate screen picker on a two-monitor setup. Confirm selected-screen persistence across restarts and disconnect fallback without crash.

10. **Analytics opt-out gate (OQ-UI-004):** Implement `UserDefaults`-gated `analyticsOptOut` flag in `MixpanelBridge`; all `track()` calls conditioned on flag. Settings UI item deferred pending OQ-001 resolution; architectural support must be present before EU distribution.

11. **Notarization credential setup:** Configure `claude-island-notarytool` Keychain profile on release engineer's machine via `xcrun notarytool store-credentials`. Document notarization credential rotation runbook (OQ-PIPE-001).

**Gate:** AC-HOOK-04 (reconnect without restart within 10 s) validated on physical hardware. Floating-pill verified on Mac mini or non-notch MacBook Pro. `testLatencySocketToPublished` passes consistently in CI. All FC-* criteria unresolved from Phases 1–3 are closed.

---

### Phase 5 — Release Pipeline Hardening and Pre-Release Checklist

**Objective:** Finalize `scripts/build.sh` and `scripts/create-release.sh` to execute all eight pipeline stages correctly. Complete release engineer checklist. Produce a signed, notarized, Sparkle-signed DMG.

**Tasks:**

1. **`scripts/build.sh` final implementation:** Stages 1–5:
   - Stage 1: `xcodebuild archive -project ClaudeIsland.xcodeproj -scheme ClaudeIsland -configuration Release -archivePath build/ClaudeIsland.xcarchive ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO MACOSX_DEPLOYMENT_TARGET=15.6 ENABLE_HARDENED_RUNTIME=YES CODE_SIGN_IDENTITY="Developer ID Application"`. Export with `method = developer-id`, `signingStyle = manual`, `--deep` for Sparkle XPC helpers.
   - Stage 2: `codesign --deep --force --options runtime` verification: `codesign -d --entitlements -` asserts `app-sandbox` absent; `codesign -dv` asserts bit `0x10000` set; `lipo -info` asserts `arm64 x86_64`.
   - Stage 3: `hdiutil create ClaudeIsland.dmg` from signed `.app`.
   - Stage 4: `xcrun notarytool submit build/ClaudeIsland.dmg --keychain-profile claude-island-notarytool --wait`; exit non-zero if status is not `Accepted`.
   - Stage 5: `xcrun stapler staple ClaudeIsland.app`; `xcrun stapler staple ClaudeIsland.dmg`; `xcrun stapler validate ClaudeIsland.dmg` must return `The validate action worked!`.

2. **`scripts/create-release.sh` final implementation:** Stages 6–8:
   - Stage 6: Assert `git ls-files .sparkle-keys/` returns empty. Set `SPARKLE_PRIVATE_KEY_PATH` from local secrets store. `SIGNATURE=$(sign_update build/ClaudeIsland.dmg "$SPARKLE_PRIVATE_KEY_PATH")`. DSA-only signatures are not used; EdDSA is mandatory (REQ-UPD-02).
   - Stage 7: `gh release create "v${VERSION}" --title "Claude Island ${VERSION}" --notes-file release-notes.md build/ClaudeIsland.dmg build/ClaudeIsland.app.zip`. Both artifacts must have passed stapler validation (stage 5) before upload. No artifact is uploaded before notarization gate.
   - Stage 8: Inject new `<item>` into `appcast.xml` with: `<title>`, `<sparkle:version>` (CFBundleVersion), `<sparkle:shortVersionString>` (CFBundleShortVersionString), `<sparkle:edSignature>` (from stage 6), `<length>`, `<url>` (GitHub Release artifact URL), `<sparkle:minimumSystemVersion>15.6</sparkle:minimumSystemVersion>`, `<pubDate>` (RFC 2822). Push to website repository. `appcast.xml` update is the final step; it must never precede passing `xcrun stapler validate` and EdDSA signature capture.

3. **Release engineer pre-release checklist verification:**
   - `git ls-files .sparkle-keys/` returns empty.
   - `codesign -d --entitlements - ClaudeIsland.app` does not include `app-sandbox = true`.
   - `codesign -dv ClaudeIsland.app` shows hardened-runtime flag.
   - `spctl -a -vvv ClaudeIsland.app` returns `source=Developer ID`.
   - `xcrun stapler validate ClaudeIsland.dmg` returns `The validate action worked!`.
   - `SPARKLE_PRIVATE_KEY_PATH` set from local secrets store.
   - `appcast.xml` `<sparkle:edSignature>` matches `sign_update` output for uploaded DMG.
   - GitHub Release artifact URLs in `appcast.xml` resolve to correct version.

4. **Sparkle private key disaster recovery design (OQ-PIPE-002):** Document transitional signing strategy (build signed with both old and new keys, or out-of-band distribution) and record in release runbook before 1.0 GA.

5. **Privacy policy prerequisite:** Confirm `https://claudeisland.com/privacy` is published before any public distribution (FU-007).

6. **`TC-BUILD-01` and `TC-RELEASE-01` CI verification:** `scripts/build.sh` produces signed `.app` with no build errors. `spctl --assess --type exec` returns acceptance for release binary. `scripts/create-release.sh` creates GitHub release and updates `appcast.xml`.

**Gate:** Full pipeline dry-run completes with exit 0 on all stages. Notarization `Accepted`. `xcrun stapler validate` passes. `appcast.xml` updated correctly. `sign_update` EdDSA signature captured and injected. `git ls-files .sparkle-keys/` returns empty before release tag.

---

### Phase 6 — 1.0 GA Readiness and Final Acceptance

**Objective:** Execute the full acceptance criteria matrix from `test:acceptance-criteria`, resolve any remaining open questions that are release-blocking, and produce the 1.0 GA artifact.

**Tasks:**

1. Run all acceptance criteria in Sections 2.1–2.12 of `test:acceptance-criteria` against the notarized DMG on macOS 15.6 Apple Silicon, Intel, and non-notch hardware.
2. Confirm AC-COMPAT-01: app launches and operates correctly on macOS 15.6 on both Apple Silicon and Intel (FC-COMPAT-01).
3. Confirm AC-SEC-01: App Sandbox disabled; hardened runtime enabled; confirmed in `.entitlements` and `xcodebuild` output.
4. Confirm AC-SEC-02: `codesign --verify --deep --strict` and `spctl --assess --type exec` pass (FC-SEC-03).
5. Confirm AC-SEC-03: Mixpanel event payloads inspected; no conversation content, file paths, or personal data present (FC-SEC-02).
6. Confirm AC-UPD-04: `.sparkle-keys/` in `.gitignore`; absent from HEAD (FC-SEC-01).
7. Resolve OQ-PIPE-001 (notarization credential rotation runbook) and OQ-PIPE-002 (Sparkle key disaster recovery) before tagging 1.0.
8. Resolve OQ-IPC-05 (socket path configurability via `CLAUDE_ISLAND_SOCKET_PATH` environment variable) before 1.0 GA to avoid permanently baking the hardcoded path into distributed hook scripts.
9. Resolve OQ-SM-003 (concurrent permission requests across sessions): design FIFO queue or session-picker UI; implement before multi-session permission correctness is claimed.
10. Confirm OQ-PIPE-003 disposition (CI notarization for nightly/staging builds): record risk acceptance or implement solution with new ADR.
11. Tag `v1.0.0`. Execute `scripts/create-release.sh`. Confirm GitHub Release artifacts, `appcast.xml` updated, Sparkle update check from a test install delivers the update.

**Gate:** All acceptance criteria pass. All failure criteria resolved. Release pipeline completes without error. 1.0 GA tagged and distributed.

---

## 3. Risks

### R-01 — Sparkle Private Key Loss or Compromise (Severity: Critical)

**Description:** If the EdDSA private key stored in `.sparkle-keys/` and the secrets manager is lost or compromised, users on old versions cannot receive a normally signed update because the public key embedded in their installed binary no longer matches a new keypair. This affects all installed instances until they self-update or reinstall manually.

**Compliance:** REQ-SEC-04, ADR-003, OQ-PIPE-002. The key must never appear in VCS (FC-SEC-01 is release-blocking). CI enforces `trufflehog` secret scan and pre-commit hooks.

**Mitigation:**
- `.sparkle-keys/` in `.gitignore` from the first commit; `git ls-files .sparkle-keys/` asserted empty before every release tag.
- Key stored in secrets manager with access control; backed up in a second offline location.
- Disaster recovery plan (OQ-PIPE-002) designed and documented before 1.0 GA: transitional build signed with both old and new keys, or out-of-band distribution mechanism.
- Key rotation runbook documented in ADR-003 addendum.

---

### R-02 — Latency Budget Violation (Severity: High)

**Description:** The 100 ms socket-to-`@Published` budget (REQ-PERF-02, FC-PERF-01) may be exceeded under main-run-loop contention, Swift async actor scheduling variance, or heavy SwiftUI diffing. The integration test `testLatencySocketToPublished` uses a 110 ms tolerance (100 ms + 10 ms), but real-world variance on an Intel machine under load may exceed this.

**Compliance:** FC-PERF-01 is release-blocking.

**Mitigation:**
- `JSONDecoder` runs on `SocketActor` before the main-actor dispatch; no JSON parsing on the main thread.
- `SessionStore.apply()` performs only O(1) dictionary operations.
- `testLatencySocketToPublished` runs in CI on both Apple Silicon and Intel targets.
- If P99 latency consistently exceeds budget on Intel, introduce a dedicated high-priority `DispatchQueue` for the main-actor dispatch hop and re-measure.

---

### R-03 — Unix Socket Reconnection Failure (Severity: High)

**Description:** `ReconnectCoordinator` must rebind within 10 seconds of socket file deletion without app restart (REQ-REL-01, FC-REL-01). A stale socket file, a permission error on rebind, or a race between the file-system monitor and the rebind attempt could cause the 10-second deadline to be missed.

**Compliance:** FC-REL-01 is release-blocking.

**Mitigation:**
- Exponential back-off schedule (250 ms → 8 s cap) implemented as specified; not parameterized without document and test updates.
- `testDisconnectAndReconnect` and `testSocketReconnectNoRestart` in `ClaudeIslandIntegrationTests` verify the full schedule.
- On rebind: remove stale socket file before `bind()`; handle `EADDRINUSE` gracefully.
- `DispatchSource` file-system event source is more reliable than polling; use it for deletion detection.

---

### R-04 — Hook Script Connectivity on macOS Without socat (Severity: High)

**Description:** `socat` is not present by default on macOS and the bash `/dev/tcp` fallback does not support `AF_UNIX`. The hook installer toggle must not ship to users until OQ-IPC-01 is resolved (detail:ipc_message_flow §4.5).

**Compliance:** OQ-IPC-01 is a pre-shipping gate for the hooks feature.

**Mitigation:**
- Implement signed, notarized Swift helper binary (`ClaudeIsland Helper`) bundled in the app. New ADR covering helper-binary signing and notarization required before hooks ship.
- Integration tests use a Swift test client directly against `UnixSocketServer` and do not depend on the shell script connectivity mechanism; tests are unblocked.
- Hook installer toggle is disabled in Settings until the helper binary ships.

---

### R-05 — App Sandbox Absent: Accidental Hook File Write Without Consent (Severity: High)

**Description:** With App Sandbox disabled, there is no OS-level boundary to prevent `HookInstaller.install()` from writing to `~/.claude/hooks/` without the consent gate. A code path that bypasses `ConsentDialogPresenter` would silently install hooks, violating REQ-HOOK-01 (FC-HOOK-01 is release-blocking).

**Compliance:** FC-HOOK-01 is release-blocking.

**Mitigation:**
- `HookInstaller.install()` throws `HookInstallerError.consentNotGranted` when `UserDefaults["hooksConsentGranted"] != true`; this is not bypassable by callers.
- `testConsentGateBlocksInstall` integration test asserts no files are written when consent is absent.
- Code review for any new call site of `HookInstaller.install()` that does not pass through `ConsentDialogPresenter`.
- CI Mixpanel and hook call-site audit extended to cover any `HookInstaller` call sites added outside `SettingsView`.

---

### R-06 — Animation Frame Drops on Intel Hardware (Severity: Medium)

**Description:** Spring animations targeting ≥ 60 fps (REQ-PERF-01) may drop frames on Intel MacBook Pro under the permission-expand path (identified as MEDIUM risk in `detail:notch_overlay_state_machine` §2.4), which animates both frame height and the permission card simultaneously.

**Compliance:** FC-PERF-02 is release-blocking.

**Mitigation:**
- `matchedGeometryEffect` prohibited inside `ExpandedContent`; fixed-frame layout mandatory for `PermissionRequestView`.
- Blur radius animated on a separate `CALayer` via a distinct `.animation(.spring, value: overlayState)` modifier to decouple from geometry layout passes.
- CI `overlay-perf-gate` step runs `xcrun xctrace` Time Profiler on Release build on both Apple Silicon and Intel.
- Pre-warm `CALayer` before the `closed → popping` transition.

---

### R-07 — Analytics Data Leakage to Mixpanel (Severity: High)

**Description:** A developer adding a new feature in `Chat/`, `Permission/`, or `IPC/` could inadvertently import `mixpanel-swift` or call `AnalyticsClient.track` with session content. FC-SEC-02 is release-blocking and any such leak constitutes a privacy violation.

**Compliance:** FC-SEC-02, REQ-SEC-05.

**Mitigation:**
- `MixpanelBridge` is the only file that imports `mixpanel-swift`; module boundaries enforced by Xcode target membership.
- CI Mixpanel call-site audit asserts zero `AnalyticsClient.track` references in `Chat/`, `Permission/`, and any file that imports `ChatMessage` or `PermissionRequest`.
- Code review checklist includes analytics isolation check for any PR touching chat or permission components.
- `testAnalyticsPayloadContainsNoSessionContent` CI step scans all `AnalyticsClient.track` call sites.

---

### R-08 — Concurrent Permission Requests Across Sessions (Severity: Medium)

**Description:** `Session.pendingPermission` is a single `PermissionRequest?`. Two concurrent Claude Code sessions each issuing a permission request simultaneously will cause the second request to overwrite `overlayState` while the first is pending. OQ-SM-003 defers the queueing strategy; until resolved, multi-session permission correctness is undefined.

**Compliance:** FC-PERM-01 is release-blocking for per-session routing but does not currently define behavior for same-overlay concurrent requests.

**Mitigation:**
- Resolve OQ-SM-003 before multi-session permission handling is claimed as supported: implement FIFO `[PermissionRequest]` queue with sequential display, or session-picker UI.
- Until resolved, document that the app supports concurrent session monitoring but serializes permission UI display to one request at a time.
- `testConcurrentTwoPaneSessions` verifies that routing is correct for distinct sessions; it does not test simultaneous display of two permission UIs.

---

### R-09 — Notarization Service Unavailability (Severity: Low)

**Description:** Notarization requires connectivity to Apple's production notarization service. If the service is unavailable during a release, the pipeline blocks at stage 4 with no fallback distribution path (REQ-REL-PIPE-01).

**Compliance:** REQ-SEC-03, REQ-REL-PIPE-01.

**Mitigation:**
- The `--wait` flag in `xcrun notarytool submit` blocks the script; a non-`Accepted` result exits non-zero and halts `create-release.sh` before any upload.
- No alternate distribution path bypasses notarization; release is deferred until the service recovers.
- Monitor Apple System Status (developer.apple.com/system-status/) before initiating a release.
- Schedule releases to avoid known Apple maintenance windows.

---

### R-10 — `@FocusState` / `canBecomeKey` Incompatibility on macOS 15.6 (Severity: Medium)

**Description:** The interaction between `canBecomeKey = false` and SwiftUI's `@FocusState` binding has not been validated on macOS 15.6 physical hardware (OQ-SM-001). If key focus cannot be acquired for `AskInputBar`, the `AskUserQuestion` feature cannot ship.

**Compliance:** OQ-SM-001 must be resolved before `AskUserQuestion` ships.

**Mitigation:**
- Test on macOS 15.6 physical hardware in Phase 4.
- If elevation to `canBecomeKey = true` is required: add `opened_waitingForInput_focused` sub-state; set `canBecomeKey = true` on entry, reset on exit; evaluate focus-steal risk; document decision and cover with test.
- If incompatible: defer `AskUserQuestion` to a post-1.0 release; ship without the AskInputBar interactive path in 1.0.
