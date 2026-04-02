---
codd:
  node_id: test:acceptance-criteria
  type: test
  depends_on:
  - id: req:claude-island-requirements
    relation: derives_from
    semantic: governance
  depended_by:
  - id: design:system-design
    relation: constrained_by
    semantic: governance
  - id: test:test-strategy
    relation: derives_from
    semantic: governance
  - id: plan:implementation-plan
    relation: constrained_by
    semantic: governance
  conventions:
  - targets:
    - test:acceptance-criteria
    reason: Every requirement (UI states, socket latency, session lifecycle, permission
      routing, non-notch fallback) must map to a verifiable criterion. Untested requirements
      block release.
  - targets:
    - module:session-monitor
    - module:ipc
    reason: Socket state updates must reflect in UI within 100 ms (REQ-PERF-02) and
      reconnection must require no app restart (REQ-REL-01); both are release-blocking
      acceptance conditions.
---

# Acceptance Criteria

## 1. Overview

This document defines the release-blocking acceptance and failure criteria for Claude Island. Every functional, performance, reliability, and security requirement in `req:claude-island-requirements` maps to a verifiable criterion here. A release may not ship until all acceptance criteria in Section 2 pass and none of the failure criteria in Section 3 are triggered.

Two invariants are non-negotiable and gate release directly:

- **REQ-PERF-02**: Socket state updates from the Unix domain socket must be reflected in the UI within 100 ms.
- **REQ-REL-01**: The app must handle Unix socket disconnection and reconnect without requiring an app restart.

These invariants apply to `module:session-monitor` (session lifecycle tracking) and `module:ipc` (Unix socket communication). Verification methods for both are called out explicitly in Section 2.

---

## 2. Acceptance Criteria

### 2.1 Notch Overlay UI

**AC-UI-01** — The overlay renders anchored to the display notch on notch-equipped MacBooks and as a floating pill at the top of the selected screen on non-notch devices. Both render paths are verified on physical hardware or a hardware-accurate simulator.

**AC-UI-02** — The overlay transitions between closed, popping, and opened states. All three states are reachable via defined triggers and return to the correct prior state on dismissal.

**AC-UI-03** — All overlay state transitions animate using spring animations at a minimum of 60 fps with no visible frame drops (verified by Instruments Time Profiler, no frame budget exceeded, REQ-PERF-01).

**AC-UI-04** — On a Mac without a notch (e.g., Mac mini, non-notch MacBook Pro), the overlay appears as a floating pill at the top of the selected screen. Crash-free operation and correct layout are confirmed.

**AC-UI-05** — The animated crab icon is visible and animating while a Claude Code session is in the `processing` phase; the animation stops when the session transitions to `idle` or `ended`.

**AC-UI-06** — An amber indicator is displayed in the overlay when a tool-permission request is pending (`waitingForInput` with a permission payload). The indicator disappears after the request is resolved.

**AC-UI-07** — A green checkmark is displayed in the overlay when a session transitions to `ended` with a success result. It does not appear on error termination.

**AC-UI-08** — The overlay bounces visually when user input is required (AskUserQuestion tool invoked or tool-permission pending) and the terminal window is not in focus.

### 2.2 Session Monitoring

**AC-MON-01** — The app monitors at least three concurrent Claude Code sessions simultaneously. All sessions display correct independent state, title, phase, and elapsed time.

**AC-MON-02** — Session lifecycle transitions follow the defined order: `idle → processing → waitingForInput → ended`. Invalid transitions (e.g., `ended → processing`) are rejected and logged without crashing.

**AC-MON-03** — A new Claude Code process started in the terminal is detected automatically and appears in the session list within 2 seconds, with no manual intervention.

**AC-MON-04** — Session title, current phase, and elapsed time are displayed per session. Elapsed time increments in real time while the session is in `processing` or `waitingForInput`.

**AC-MON-05** — Invoking `/clear` in the terminal causes the corresponding session's state to be cleared in the overlay within 100 ms (REQ-PERF-02).

**AC-MON-06** — No audio notification and no visual bounce or amber indicator is shown while the terminal window associated with the session has keyboard focus.

**AC-MON-PERF** — End-to-end latency from socket message delivery to UI state update is measured at ≤ 100 ms under normal load (REQ-PERF-02). This is verified by timestamping the socket write and the subsequent `@Published` property update in `module:session-monitor`, with a tolerance of ±10 ms.

### 2.3 Hook Integration

**AC-HOOK-01** — On first launch, with explicit user consent, hook scripts are installed into `~/.claude/hooks/`. Without consent, no files are written.

**AC-HOOK-02** — Hooks communicate session events to the app over a Unix domain socket. At minimum, session-start, tool-call, tool-result, and session-end events are delivered and parsed correctly.

**AC-HOOK-03** — The hooks installer toggle in Settings enables and disables hook installation. Toggling off removes or disables the hooks; toggling on reinstalls them. The toggle state persists across app restarts.

**AC-HOOK-04 (REQ-REL-01)** — When the hook process restarts or the socket connection is dropped, the app detects the disconnection, attempts reconnection with exponential back-off, and restores full session monitoring within 10 seconds — without requiring an app restart. This is verified by killing the socket peer and confirming automatic reconnection in `module:ipc`.

### 2.4 Tool Permission Approvals

**AC-PERM-01** — When a Claude Code session requests tool approval, the notch overlay expands and displays the tool name and the complete tool input within 100 ms of socket message receipt (REQ-PERF-02).

**AC-PERM-02** — Tapping Approve or Deny in the overlay sends the correct response back to the Claude Code process via the Unix socket. The process receives the response and continues execution.

**AC-PERM-03** — The approval/denial response is delivered over the Unix socket to the correct Claude Code process. No response is delivered to a different session.

**AC-PERM-04** — If the terminal is not visible when a permission request arrives, the overlay opens automatically.

**AC-PERM-05** — For sessions running inside tmux, the approval response is routed to the correct tmux pane. Routing is verified by confirming the target pane receives the response when at least two tmux panes each host a Claude Code session simultaneously.

### 2.5 Chat History

**AC-CHAT-01** — Full conversation history is parsed from JSONL files and displayed in the chat view for each session. All user and assistant message types present in the JSONL are rendered.

**AC-CHAT-02** — Markdown formatting (bold, inline code, fenced code blocks, unordered and ordered lists) is rendered correctly using the `apple/swift-markdown` library.

**AC-CHAT-03** — The chat view auto-scrolls to the latest message when a new message arrives. Manually scrolling up pauses auto-scroll; a scroll-to-bottom affordance resumes it.

**AC-CHAT-04** — Tool calls are shown inline in the conversation with their name, arguments, and completion status (pending, success, or failure).

**AC-CHAT-05** — Switching between sessions and returning to a previous session does not cause render flicker. Cached history is displayed immediately without re-parsing.

**AC-CHAT-06** — When a session is in `waitingForInput` (AskUserQuestion), an interactive prompt bar is displayed. Submitting a response from the bar relays the text to the Claude Code process.

**AC-CHAT-07** — When a session is in `idle`, a message input bar is available. Submitting from the bar delivers the message to the Claude Code session.

### 2.6 AskUserQuestion Support

**AC-ASK-01** — The AskUserQuestion tool invocation is detected via the Unix socket and the prompt text is surfaced in the chat view immediately.

**AC-ASK-02** — The user can type and submit a response from within the app. The response is delivered to the Claude Code session without the user switching to the terminal.

### 2.7 Multi-Monitor Support

**AC-SCREEN-01** — The Settings panel includes a screen picker listing all connected displays. Selecting a display moves the overlay to that display.

**AC-SCREEN-02** — On first launch, the overlay defaults to the built-in display when one is present.

**AC-SCREEN-03** — The selected display identifier persists across app restarts. On relaunch, the overlay appears on the previously selected screen.

**AC-SCREEN-04** — If the previously selected screen is disconnected at launch or during runtime, the app falls back to an available screen without crashing. A new selection can be made from Settings.

### 2.8 Sound Notifications

**AC-SOUND-01** — A configurable system sound plays when a Claude Code session transitions to `ended` with success. The sound is the one currently selected in Settings.

**AC-SOUND-02** — The sound picker in Settings lists at least 14 system sounds. All listed sounds are playable.

**AC-SOUND-03** — The sound picker is an inline scrollable list within the Settings menu (not a modal or separate window).

**AC-SOUND-04** — The selected sound persists across app restarts.

**AC-SOUND-05** — No sound plays when the terminal window associated with the completing session is currently in focus.

### 2.9 Settings

**AC-SET-01** — The Settings panel is accessible from the notch overlay without navigating to the menu bar icon.

**AC-SET-02** — The Settings panel contains all six items: screen picker, inline scrollable sound picker, launch-at-login toggle, hooks installer toggle, accessibility permissions status with a grant-access guide, and auto-update control.

**AC-SET-03** — The launch-at-login toggle uses `SMAppService`. Enabling it causes the app to launch automatically on the next login; disabling it removes the login item.

**AC-SET-04** — When accessibility permissions are not granted, the Settings panel displays the missing-permissions status and provides a direct path to System Settings > Privacy & Security > Accessibility.

### 2.10 Auto-Update

**AC-UPD-01** — The app fetches the update feed from `https://claudeisland.com/appcast.xml` via the Sparkle framework and installs available updates.

**AC-UPD-02** — Updates are verified against an EdDSA signature before installation. An update with an invalid or missing signature is rejected and the user is notified.

**AC-UPD-03** — When an unseen update is available, the Settings menu displays a green dot indicator on the auto-update control.

**AC-UPD-04** — Sparkle private keys are absent from the repository. The `.sparkle-keys/` directory is confirmed to be listed in `.gitignore` and absent from the current HEAD.

### 2.11 Build, Security, and Release

**AC-SEC-01** — App Sandbox is disabled in the release entitlements to allow Unix socket and file-system access. Hardened runtime is enabled. Both are confirmed in the `.entitlements` file and `xcodebuild` output.

**AC-SEC-02** — Release binaries pass `codesign --verify --deep --strict` and `spctl --assess --type exec` (notarization check).

**AC-SEC-03** — Analytics events are limited to app-lifecycle events (launch, session start). No conversation content, file paths, or personal data appear in Mixpanel event payloads, verified by inspecting the Mixpanel SDK call sites.

**AC-BUILD-01** — `scripts/build.sh` produces a signed `.app` bundle using the Developer ID distribution certificate and hardened runtime with no build errors.

**AC-RELEASE-01** — The DMG is notarized and signed with the Sparkle EdDSA key. `scripts/create-release.sh` creates a GitHub release and updates `appcast.xml`.

**AC-COMPAT-01** — The app launches and operates correctly on macOS 15.6 on both Apple Silicon and Intel, and on machines without a notch.

### 2.12 Testing Requirements

**AC-TEST-01** — A unit test target exists in the Xcode project.

**AC-TEST-02** — All `SessionPhase` state machine transitions (`idle → processing`, `processing → waitingForInput`, `waitingForInput → ended`, `processing → ended`, `idle → ended`) have passing unit tests.

**AC-TEST-03** — Hook installation and Unix socket communication logic are covered by integration tests. Tests exercise at minimum: socket connect, event send, event receive, and socket disconnect-and-reconnect.

---

## 3. Failure Criteria

The following conditions constitute release-blocking failures. Any one of them prevents shipment.

**FC-PERF-01** — Socket-to-UI latency exceeds 100 ms under normal operating conditions (single session, idle system). Violates REQ-PERF-02 and the `module:ipc` / `module:session-monitor` invariant.

**FC-PERF-02** — The overlay animation drops below 60 fps on a supported machine (Apple Silicon MacBook with notch). Violates REQ-PERF-01.

**FC-PERF-03** — Parsing a JSONL chat history file blocks the main thread and causes the UI to freeze or miss a frame during parsing. Violates REQ-PERF-03.

**FC-REL-01** — The app requires a restart after a Unix socket disconnection to resume session monitoring. Violates REQ-REL-01 and the `module:ipc` reconnection invariant.

**FC-REL-02** — A missing or corrupt JSONL file causes a crash. The chat view must show an empty state or an error state instead. Violates REQ-REL-02.

**FC-REL-03** — A display disconnection event causes a crash. Violates REQ-REL-03.

**FC-UI-01** — The non-notch fallback (floating pill) is absent or crashes on a Mac without a notch. Violates REQ-UI-04 and REQ-COMPAT-03.

**FC-SEC-01** — Any Sparkle private key material appears in a git commit, staged file, or build artifact. Violates REQ-SEC-04.

**FC-SEC-02** — Any conversation content, file path, or personally identifiable data is transmitted to Mixpanel. Violates REQ-SEC-05.

**FC-SEC-03** — A release binary fails notarization (`spctl` assessment returns rejection). Violates REQ-SEC-03 and REQ-REL-PIPE-01.

**FC-SEC-04** — Hardened runtime is absent from a release build. Violates REQ-SEC-02.

**FC-PERM-01** — An approval or denial response is delivered to the wrong Claude Code session or tmux pane. Violates REQ-PERM-03 and REQ-PERM-05.

**FC-HOOK-01** — Hooks are written to `~/.claude/hooks/` without explicit user consent. Violates REQ-HOOK-01.

**FC-TEST-01** — The unit test target does not exist or `SessionPhase` transitions have no automated coverage. Violates REQ-TEST-01 and REQ-TEST-02.

**FC-TEST-02** — Hook installation or socket communication logic has no integration test coverage. Violates REQ-TEST-03.

**FC-COMPAT-01** — The app fails to launch or crashes on macOS 15.6 on either Apple Silicon or Intel. Violates REQ-COMPAT-01 and REQ-COMPAT-02.
