---
codd:
  node_id: design:system-design
  type: design
  depends_on:
  - id: test:acceptance-criteria
    relation: constrained_by
    semantic: governance
  - id: governance:decisions
    relation: constrained_by
    semantic: governance
  depended_by:
  - id: design:ui-design
    relation: depends_on
    semantic: technical
  - id: design:ipc-hook-design
    relation: depends_on
    semantic: technical
  - id: design:session-lifecycle-design
    relation: depends_on
    semantic: technical
  - id: design:release-pipeline-design
    relation: depends_on
    semantic: technical
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:system-design
    reason: Architecture must reflect App Sandbox disabled and hardened runtime enabled.
      macOS 15.6+ on Apple Silicon and Intel is the minimum supported platform (REQ-COMPAT-01,
      REQ-COMPAT-02).
  - targets:
    - design:system-design
    reason: Anonymous telemetry only; no conversation content or personal data may
      be transmitted (REQ-SEC-05). The analytics subsystem boundary must be explicit
      in the system design.
  - targets:
    - design:system-design
    reason: All third-party dependencies (Sparkle, swift-markdown, mixpanel-swift)
      must be resolved via Swift Package Manager (REQ-BUILD-01). No other dependency
      managers are permitted.
---

# System Design Overview

## 1. Overview

Claude Island is a native macOS menu bar application that renders a Dynamic Island-style notch overlay for monitoring Claude Code CLI sessions in real time. It targets macOS 15.6 and later on both Apple Silicon and Intel (Universal Binary, REQ-COMPAT-01, REQ-COMPAT-02) and is distributed exclusively as a signed, notarized DMG via GitHub Releases and direct download — not through the Mac App Store.

The application bridges two worlds: the Claude Code CLI (running in terminal emulators and tmux panes) and a persistent ambient UI layer that surfaces session state, tool-permission requests, chat history, and response input without requiring the user to switch windows. All IPC between hook scripts and the app traverses a Unix domain socket at `~/.claude/claude-island.sock`. Hook scripts installed into `~/.claude/hooks/` emit newline-delimited JSON events on session start, tool call, tool result, and session end; the app parses those events, updates its in-memory session model, and drives the SwiftUI overlay.

**Release-blocking constraints enforced by this design:**

| Constraint | Requirement | Control |
|---|---|---|
| App Sandbox disabled | REQ-SEC-01 / ADR-001 | Entitlements file; CI assertion; release checklist |
| Hardened runtime enabled | REQ-SEC-02 / ADR-002 | `ENABLE_HARDENED_RUNTIME = YES`; CI codesign flag check |
| Notarization before distribution | REQ-SEC-03 / ADR-004 | `xcrun notarytool submit --wait`; stapler validate gate |
| Sparkle EdDSA keys never in VCS | REQ-SEC-04 / ADR-003 | `.gitignore`; CI secret scan (trufflehog); pre-commit hook |
| Anonymous telemetry only | REQ-SEC-05 / ADR-008 | Analytics subsystem boundary; Mixpanel call-site audit |
| All dependencies via SPM | REQ-BUILD-01 | `Package.swift` sole dependency manifest; no CocoaPods, no Carthage |
| macOS 15.6+ minimum | REQ-COMPAT-01 / ADR-009 | `MACOSX_DEPLOYMENT_TARGET = 15.6` |
| Universal Binary | REQ-COMPAT-02 / ADR-009 | `ARCHS = arm64 x86_64`; `ONLY_ACTIVE_ARCH = NO` for Release |
| Socket-to-UI latency ≤ 100 ms | REQ-PERF-02 | Async socket reader on background actor; `@Published` on main actor |
| Automatic reconnection without restart | REQ-REL-01 | Exponential back-off reconnect loop in `module:ipc` |
| Overlay animation ≥ 60 fps | REQ-PERF-01 | Spring animations; Instruments Time Profiler gate in CI |

---

## 2. Architecture

### 2.1 Layer Overview

The system is structured into five layers with explicit boundaries:

```
┌─────────────────────────────────────────────┐
│              Presentation Layer              │
│  NotchOverlayView · ChatHistoryView          │
│  PermissionRequestView · SettingsView        │
│  SessionListView · SoundPickerView           │
├─────────────────────────────────────────────┤
│            Session Model Layer               │
│  SessionStore · SessionPhase state machine   │
│  ChatHistoryParser · PermissionRouter        │
├─────────────────────────────────────────────┤
│              IPC Layer (module:ipc)          │
│  UnixSocketServer · MessageFramer            │
│  ReconnectCoordinator (exp. back-off)        │
├─────────────────────────────────────────────┤
│         Integration & Services Layer         │
│  HookInstaller · SMAppService (login item)   │
│  SparkleUpdater · ScreenSelector             │
│  SoundPlayer (AVFoundation)                  │
├─────────────────────────────────────────────┤
│           Analytics Subsystem (isolated)     │
│  AnalyticsClient · MixpanelBridge            │
│  (app_launched, session_started events only) │
└─────────────────────────────────────────────┘
```

The Analytics Subsystem is intentionally isolated behind `AnalyticsClient`, a protocol that the rest of the application calls. `MixpanelBridge` is the sole conforming type and the sole location where the Mixpanel SDK is imported. No other module imports `mixpanel-swift`. This boundary enforces REQ-SEC-05: conversation content, file paths, tool names, and personally identifiable data cannot reach Mixpanel because no other code path has access to the SDK. Mixpanel call sites are audited at release time against AC-SEC-03.

### 2.2 Third-Party Dependencies (Swift Package Manager)

All third-party dependencies are resolved exclusively via Swift Package Manager (REQ-BUILD-01). No other dependency managers (CocoaPods, Carthage, manual xcframework embedding) are permitted.

| Package | Source | Purpose |
|---|---|---|
| `Sparkle` | `https://github.com/sparkle-project/Sparkle` | Over-the-air updates with EdDSA verification (ADR-006) |
| `apple/swift-markdown` | `https://github.com/apple/swift-markdown` | Markdown rendering in `ChatHistoryView` (AC-CHAT-02) |
| `mixpanel-swift` | `https://github.com/mixpanel/mixpanel-swift` | Anonymous telemetry (ADR-008); isolated behind `AnalyticsClient` |

`Package.resolved` is committed to version control. Dependency updates are reviewed manually; each dependency must provide a Universal Binary xcframework or compile from source as Universal (ADR-009).

### 2.3 Notch Overlay UI

The overlay is a borderless, non-activating `NSWindow` set to `NSWindow.Level.statusBar + 1`. On notch-equipped MacBooks, the window is anchored to the display notch region. On non-notch hardware (Mac mini, non-notch MacBook Pro, external displays), the overlay renders as a floating pill at the top of the selected screen — this fallback is required by REQ-UI-04 and REQ-COMPAT-03 and its absence constitutes failure criterion FC-UI-01.

**Overlay states and transitions:**

```
closed ──tap notch──▶ popping ──animation complete──▶ opened
opened ──dismiss──────────────────────────────────────▶ closed
opened ──permission arrives──▶ opened (expanded, amber indicator)
opened ──session ends/success──▶ opened (green checkmark, 3 s) ──▶ closed
```

All transitions use SwiftUI spring animations targeting ≥ 60 fps (REQ-PERF-01; FC-PERF-02 is the release-blocking failure). The animated crab icon is active during the `processing` phase; it halts on `idle` or `ended` (AC-UI-05). The overlay bounces when user input is required and the terminal window does not have keyboard focus (AC-UI-08).

The overlay does not activate the application (`NSApp.activate`) and does not steal keyboard focus from the terminal.

### 2.4 IPC Layer — module:ipc

The app creates a Unix domain socket at `~/.claude/claude-island.sock` at launch and listens for incoming connections. Each active Claude Code session connects as a separate client (supporting ≥ 3 concurrent sessions, AC-MON-01). Messages are newline-delimited JSON frames.

**Socket lifecycle:**

1. App launches → `UnixSocketServer` binds and listens on `~/.claude/claude-island.sock`.
2. Hook script connects, sends event JSON, optionally awaits a response (tool approval).
3. App parses event, dispatches to `SessionStore` on the main actor.
4. For tool-permission events, `PermissionRouter` holds the connection open until the user approves or denies via the overlay.
5. The approval/denial JSON is written back to the exact client connection that sent the request (AC-PERM-03; FC-PERM-01 is release-blocking).

**Reconnection (REQ-REL-01 / AC-HOOK-04):**

`ReconnectCoordinator` monitors socket health. On disconnection it initiates exponential back-off reconnect attempts (starting at 250 ms, doubling to a cap of 8 s). Full session monitoring is restored within 10 seconds. The app must never require a restart for reconnection; FC-REL-01 is a release-blocking failure criterion.

**Latency (REQ-PERF-02):**

The socket reader runs on a dedicated background actor (`SocketActor`). On receipt of a complete frame, it publishes to `SessionStore` which updates `@Published` properties on the main actor. End-to-end latency from socket write to `@Published` update must be ≤ 100 ms (AC-MON-PERF, FC-PERF-01). Verified by timestamping the socket write and the `@Published` update, tolerance ±10 ms.

### 2.5 Session Model Layer — module:session-monitor

`SessionStore` is an `ObservableObject` that holds a dictionary of `Session` values keyed by session ID. `Session` contains:

- `id: UUID`
- `phase: SessionPhase` (`idle | processing | waitingForInput | ended`)
- `title: String`
- `elapsedTime: TimeInterval` (increments via a `Timer` while `processing` or `waitingForInput`)
- `pendingPermission: PermissionRequest?`
- `tmuxPane: String?`

**`SessionPhase` state machine:**

Valid transitions: `idle → processing`, `processing → waitingForInput`, `waitingForInput → ended`, `processing → ended`, `idle → ended`. All five transitions have passing unit tests (AC-TEST-02, REQ-TEST-02). Invalid transitions (e.g., `ended → processing`) are rejected, logged via `os_log`, and do not crash (AC-MON-02).

New Claude Code processes detected within 2 seconds of launch (AC-MON-03). The `/clear` terminal command clears the corresponding session within 100 ms (AC-MON-05).

### 2.6 Chat History

`ChatHistoryParser` reads JSONL files from the Claude Code session directory. Parsing runs off the main thread; a corrupt or missing file produces an empty/error state without crashing (FC-REL-02). Parsed history is cached per session; switching between sessions displays cached history without re-parsing or flicker (AC-CHAT-05).

`ChatHistoryView` renders messages using `apple/swift-markdown` for Markdown formatting (bold, inline code, fenced code blocks, lists — AC-CHAT-02). Tool calls render inline with name, arguments, and status (pending / success / failure — AC-CHAT-04). The view auto-scrolls to the latest message; manual scroll-up pauses auto-scroll; a scroll-to-bottom affordance resumes it (AC-CHAT-03).

JSONL parsing must not block the main thread (REQ-PERF-03; FC-PERF-03 is release-blocking).

### 2.7 Tool Permission Routing

When a Claude Code session emits a tool-permission event, `PermissionRouter` routes the request to the correct session. The overlay expands and displays the tool name and complete tool input within 100 ms of socket message receipt (AC-PERM-01).

For sessions running inside tmux, the approval response is delivered to the correct tmux pane identified by the `tmuxPane` field in the session record (AC-PERM-05). Routing correctness is verified with two concurrent tmux panes each hosting a separate Claude Code session. Misrouting constitutes FC-PERM-01, a release-blocking failure.

If the terminal is not visible when a permission request arrives, the overlay opens automatically (AC-PERM-04).

### 2.8 AskUserQuestion Support

`AskUserQuestion` tool invocations are detected from the socket stream and surfaced as `waitingForInput` phase with a `promptText` payload. An interactive prompt bar appears in `ChatHistoryView` (AC-CHAT-06, AC-ASK-01). The user types and submits a response; `module:ipc` writes the response to the originating socket connection (AC-ASK-02). The amber indicator and overlay bounce activate for the duration of the wait (AC-UI-06, AC-UI-08).

### 2.9 Multi-Monitor Support

`ScreenSelector` wraps `NSScreen.screens` and persists the selected screen identifier in `UserDefaults`. On first launch the app defaults to the built-in display (AC-SCREEN-02). If the persisted display is disconnected at launch or during runtime, `ScreenSelector` falls back to the first available screen without crashing (AC-SCREEN-04; FC-REL-03 is release-blocking). The Settings panel provides a screen picker (AC-SCREEN-01, AC-SET-02).

### 2.10 Sound Notifications

`SoundPlayer` uses `AVFoundation` to play system sounds. A configurable sound plays when a session transitions to `ended` with success (AC-SOUND-01). No sound plays when the terminal window hosting the completing session is in focus (AC-SOUND-05). The Settings panel includes an inline scrollable sound picker listing ≥ 14 system sounds (AC-SOUND-02, AC-SOUND-03). The selection persists in `UserDefaults` (AC-SOUND-04).

### 2.11 Settings Panel

The Settings panel is accessible from the notch overlay without navigating to the menu bar icon (AC-SET-01). It contains exactly six items (AC-SET-02):

1. **Screen picker** — select the target display (AC-SCREEN-01).
2. **Inline scrollable sound picker** — select the completion sound (AC-SOUND-02, AC-SOUND-03).
3. **Launch-at-login toggle** — backed by `SMAppService` (AC-SET-03).
4. **Hooks installer toggle** — enables/disables hook installation to `~/.claude/hooks/` (AC-HOOK-03). Toggling on requires explicit user consent before writing any files (AC-HOOK-01; FC-HOOK-01 is release-blocking).
5. **Accessibility permissions status** — shows grant status; provides a direct path to System Settings > Privacy & Security > Accessibility (AC-SET-04).
6. **Auto-update control** — Sparkle-backed; displays a green dot indicator when an unseen update is available (AC-UPD-03).

### 2.12 Hook Installation

`HookInstaller` writes scripts to `~/.claude/hooks/` only after the user explicitly enables the hooks installer toggle in Settings (AC-HOOK-01). Without consent no files are written (FC-HOOK-01). On toggle-off, the hooks are removed or disabled. The toggle state persists in `UserDefaults` across restarts (AC-HOOK-03).

Hooks communicate with the app over the Unix domain socket (ADR-005). At minimum, session-start, tool-call, tool-result, and session-end events are delivered and parsed (AC-HOOK-02). Integration tests cover socket connect, event send, event receive, and disconnect-and-reconnect (AC-TEST-03, REQ-TEST-03).

### 2.13 Auto-Update Subsystem

Sparkle is integrated via SPM and configured with:

- `SUFeedURL` = `https://claudeisland.com/appcast.xml`
- `SUPublicEDKey` = Ed25519 public key embedded in `Info.plist`

Updates are verified against an EdDSA signature before installation (AC-UPD-02). Updates with invalid or missing signatures are rejected and the user is notified. Sparkle's XPC-based installer is compatible with hardened runtime (ADR-002). The Sparkle EdDSA private key resides exclusively in `.sparkle-keys/` (gitignored) and in the secrets manager (ADR-003); it is referenced in `scripts/create-release.sh` via the environment variable `SPARKLE_PRIVATE_KEY_PATH`.

### 2.14 Analytics Subsystem Boundary

The `AnalyticsClient` protocol is the sole entry point for telemetry across the entire application. Its two permitted events are:

- `app_launched` — fired once at startup with no additional properties.
- `session_started` — fired when a new session is detected, with no properties identifying user, project, content, or file paths.

`MixpanelBridge` is the sole conforming type and the sole import site of `mixpanel-swift`. The Mixpanel project token is a non-secret value embedded in the binary. No device fingerprint, IP address correlation, or user ID is persisted (ADR-008). Compliance with REQ-SEC-05 is verified at release time by auditing all `AnalyticsClient` call sites and confirming no conversation content, tool names, or file paths appear in any event payload (AC-SEC-03; FC-SEC-02 is release-blocking).

A privacy policy describing this anonymous telemetry is required at `https://claudeisland.com/privacy` before public distribution (FU-007).

### 2.15 Build, Signing, and Release Pipeline

**Entitlements (release-blocking):**

- `com.apple.security.app-sandbox` = `false` (ADR-001; AC-SEC-01). CI asserts this key is absent or false before code signing.
- Hardened runtime enabled via `ENABLE_HARDENED_RUNTIME = YES` (ADR-002; AC-SEC-01). CI asserts `codesign -dv` reports the hardened-runtime flag (bit `0x10000`).
- No JIT, no library-validation disable, no dyld-environment-variable entitlements without a new ADR.

**Signing:**

- `CODE_SIGN_IDENTITY = "Developer ID Application"` for Release and Distribution configurations (ADR-007).
- `scripts/build.sh` passes `--deep` to `codesign` for embedded Sparkle XPC helpers.

**Notarization (release-blocking):**

- `scripts/build.sh` calls `xcrun notarytool submit --keychain-profile claude-island-notarytool --wait`. A non-`Accepted` status exits non-zero and halts the pipeline (ADR-004; AC-SEC-02; FC-SEC-03).
- `xcrun stapler staple` is called for both `ClaudeIsland.app` and `ClaudeIsland.dmg`.
- `xcrun stapler validate ClaudeIsland.dmg` must return `The validate action worked!` before upload.

**Secret scanning (ADR-003):**

- `.sparkle-keys/` is listed in `.gitignore`.
- CI runs `trufflehog` (or equivalent) as a required status check on every pull request. Detection blocks merge (FU-002).
- A pre-commit hook asserts no file matching `*.key`, `*.pem`, `*.p8`, `private_key`, or `sparkle_private*` is staged.

**Platform:**

- `MACOSX_DEPLOYMENT_TARGET = 15.6`; `ARCHS = arm64 x86_64`; `ONLY_ACTIVE_ARCH = NO` for Release (ADR-009).
- All three SPM dependencies must build as Universal Binary.

### 2.16 Non-Functional Requirements Summary

| Requirement | Threshold | Failure criterion |
|---|---|---|
| Socket-to-UI latency | ≤ 100 ms (±10 ms tolerance) | FC-PERF-01 |
| Overlay animation frame rate | ≥ 60 fps | FC-PERF-02 |
| JSONL parse blocking main thread | Must not block | FC-PERF-03 |
| Reconnection after socket drop | ≤ 10 s, no restart | FC-REL-01 |
| Missing/corrupt JSONL | Empty/error state, no crash | FC-REL-02 |
| Display disconnection | Fallback to available screen, no crash | FC-REL-03 |
| New session detection latency | ≤ 2 s | AC-MON-03 |

### 2.17 Testing Architecture

A unit test target exists in `ClaudeIsland.xcodeproj` (AC-TEST-01, REQ-TEST-01). The minimum test coverage includes:

- All five `SessionPhase` state machine transitions with passing unit tests (AC-TEST-02).
- Integration tests for hook installation: socket connect, event send, event receive, and socket disconnect-and-reconnect (AC-TEST-03).

Absence of the unit test target or absence of `SessionPhase` transition coverage constitutes FC-TEST-01. Absence of hook/socket integration tests constitutes FC-TEST-02. Both are release-blocking.

---

## 3. Open Questions

**OQ-001 — Telemetry Opt-Out Mechanism (FU-003)**
ADR-008 defers the opt-out decision. If the app is distributed to users in jurisdictions where anonymous telemetry without an explicit opt-out violates applicable law (e.g., GDPR), an opt-out preference must be added to the Settings panel and must be checked before any `AnalyticsClient` event is sent. The current architecture supports this by gating `MixpanelBridge` calls behind a `UserDefaults` flag, but the setting does not yet exist in the Settings panel. This must be resolved before EU distribution.

**OQ-002 — Notarization Credentials Rotation Runbook (FU-004)**
The procedure for revoking and regenerating the app-specific password for the `claude-island-notarytool` Keychain profile, and verifying notarization still succeeds after rotation, is not yet documented in the release runbook. This is required before 1.0 GA.

**OQ-003 — Sparkle Private Key Disaster Recovery (FU-005)**
The mechanism for distributing a key-rotation build to users who are running an old version and cannot receive a normally signed update (because the old public key is no longer valid) is not yet defined. This requires a forced-migration strategy — either a transitional update signed with both old and new keys, or a side-channel distribution — and must be documented before 1.0 GA.

**OQ-004 — Intel Deprecation Threshold (FU-006)**
ADR-009 defers dropping Intel support. The trigger condition (proposed: fewer than 5% of `app_launched` telemetry events from Intel devices for two consecutive quarters) should be formally recorded and reviewed at Q4 2026. A new ADR will be required to execute the deprecation.

**OQ-005 — App Sandbox Re-evaluation (FU-008)**
ADR-001 permanently disables App Sandbox based on current macOS entitlement capabilities. If Apple introduces entitlements or APIs that allow Unix socket access to arbitrary paths from a sandboxed app, ADR-001 should be revisited. Flag for re-evaluation at macOS 17.

**OQ-006 — Hook Script Distribution and Versioning**
The hook scripts installed into `~/.claude/hooks/` are bundled with the app. There is no defined versioning scheme for distinguishing hook script versions from app versions, no migration strategy for upgrading hook scripts in place when the app updates via Sparkle, and no specification for how the installer handles pre-existing hook scripts from a previous installation. This must be resolved before the hooks installer toggle ships.

**OQ-007 — tmux Pane Routing Implementation**
AC-PERM-05 requires that approval responses be delivered to the correct tmux pane when multiple Claude Code sessions run in separate panes. The mechanism for identifying pane targets (e.g., via `$TMUX_PANE` environment variable in the hook script, embedded in the event JSON) is not yet formally specified. The `tmuxPane` field in the `Session` model is a placeholder. Misrouting is FC-PERM-01 (release-blocking), so the routing protocol must be finalized and covered by integration tests before release.
