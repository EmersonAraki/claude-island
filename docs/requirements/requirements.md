---
codd:
  node_id: "req:claude-island-requirements"
  type: requirement
  status: approved
  confidence: 0.95
---

# Claude Island — Product Requirements

## 1. Product Overview

Claude Island is a native macOS menu bar application that surfaces Claude Code CLI session activity in a Dynamic Island-style notch overlay. It gives developers real-time visibility into running Claude Code sessions, inline tool-permission approvals, and conversation history — all without switching away from their current application.

**Target users:** macOS developers running Claude Code CLI  
**Platform:** macOS 15.6+, Apple Silicon and Intel (notch and non-notch devices)  
**Distribution:** Direct download (DMG), auto-update via Sparkle  
**License:** Apache 2.0

---

## 2. Core Functional Requirements

### 2.1 Notch Overlay UI

- REQ-UI-01: The app shall render a Dynamic Island-style overlay anchored to the macOS display notch.
- REQ-UI-02: The overlay shall support three states: closed (minimal footprint), popping (brief animation), and opened (expanded content).
- REQ-UI-03: The overlay shall animate smoothly between states using spring animations.
- REQ-UI-04: On devices without a notch the overlay shall render as a floating pill at the top of the selected screen.
- REQ-UI-05: A crab icon shall animate during active processing to indicate session activity.
- REQ-UI-06: The overlay shall display an amber indicator when a tool-permission request is pending.
- REQ-UI-07: The overlay shall display a green checkmark when a session completes successfully.
- REQ-UI-08: The overlay shall bounce to attract attention when user input is required.

### 2.2 Session Monitoring

- REQ-MON-01: The app shall monitor multiple concurrent Claude Code sessions simultaneously.
- REQ-MON-02: Session state shall be tracked through a defined lifecycle: idle → processing → waitingForInput → ended.
- REQ-MON-03: The app shall detect new Claude Code processes and begin monitoring them automatically.
- REQ-MON-04: Session titles, current phase, and elapsed time shall be displayed per session.
- REQ-MON-05: The app shall clear session state when `/clear` is invoked in the terminal.
- REQ-MON-06: The app shall suppress audio and visual notifications when the associated terminal window is in focus.

### 2.3 Hook Integration

- REQ-HOOK-01: The app shall install hooks into `~/.claude/hooks/` on first launch with user consent.
- REQ-HOOK-02: Hooks shall communicate session events to the app over a Unix domain socket.
- REQ-HOOK-03: The hooks installer toggle in Settings shall allow the user to enable or disable hook installation at any time.
- REQ-HOOK-04: The app shall handle socket reconnection gracefully if the hook process restarts.

### 2.4 Tool Permission Approvals

- REQ-PERM-01: When a Claude Code session requests tool approval, the app shall expand the notch overlay and present the tool name and input.
- REQ-PERM-02: The user shall be able to approve or deny the tool request directly from the notch overlay.
- REQ-PERM-03: Approve/deny responses shall be relayed back to the Claude Code process via the Unix socket.
- REQ-PERM-04: If the terminal is not visible, the overlay shall open automatically on a permission request.
- REQ-PERM-05: The app shall support tmux session matching to route approvals to the correct pane.

### 2.5 Chat History

- REQ-CHAT-01: The app shall parse and display full conversation history for each session from JSONL files.
- REQ-CHAT-02: Chat messages shall be rendered with Markdown formatting (bold, code, lists, etc.).
- REQ-CHAT-03: The chat view shall auto-scroll to the latest message and support pausing scroll.
- REQ-CHAT-04: Tool calls and their completion status shall be visible inline in the conversation.
- REQ-CHAT-05: The app shall cache parsed chat history in memory to prevent render flicker on view transitions.
- REQ-CHAT-06: An interactive prompt bar shall appear when the session is waiting for user input (AskUserQuestion tool).
- REQ-CHAT-07: A message input bar shall be available when the session is idle.

### 2.6 AskUserQuestion Support

- REQ-ASK-01: The app shall detect when Claude Code invokes the AskUserQuestion tool and surface the prompt in the chat view.
- REQ-ASK-02: The user shall be able to type and submit a response from within the app without switching to the terminal.

### 2.7 Multi-Monitor Support

- REQ-SCREEN-01: The user shall be able to select which display the notch overlay appears on via Settings.
- REQ-SCREEN-02: The app shall default to the built-in display when available.
- REQ-SCREEN-03: Screen selection shall persist across app restarts using a stable display identifier.
- REQ-SCREEN-04: If the previously selected screen is disconnected, the app shall fall back gracefully to an available screen.

### 2.8 Sound Notifications

- REQ-SOUND-01: The app shall play a configurable system sound when a Claude Code session completes.
- REQ-SOUND-02: A minimum of 14 system sounds shall be available for selection.
- REQ-SOUND-03: The sound picker shall be an inline scrollable list within the Settings menu.
- REQ-SOUND-04: Sound selection shall persist across app restarts.
- REQ-SOUND-05: No sound shall play when the associated terminal window is currently focused.

### 2.9 Settings

- REQ-SET-01: The app shall provide a menu bar settings panel accessible from the notch overlay.
- REQ-SET-02: Settings shall include: screen picker, sound picker, launch-at-login toggle, hooks installer toggle, accessibility permissions status, and auto-update control.
- REQ-SET-03: Launch at login shall be managed via `SMAppService`.
- REQ-SET-04: An accessibility permissions check shall be displayed and guide the user to grant access when missing.

### 2.10 Auto-Update

- REQ-UPD-01: The app shall support automatic over-the-air updates via the Sparkle framework.
- REQ-UPD-02: Updates shall be signed with EdDSA keys and verified before installation.
- REQ-UPD-03: The Settings menu shall indicate when an unseen update is available (green dot indicator).
- REQ-UPD-04: The app shall fetch the update feed from `https://claudeisland.com/appcast.xml`.

---

## 3. Non-Functional Requirements

### 3.1 Performance

- REQ-PERF-01: The notch overlay shall animate at 60 fps with no visible jank.
- REQ-PERF-02: Session state updates from the Unix socket shall be reflected in the UI within 100 ms.
- REQ-PERF-03: Parsing JSONL chat history files shall not block the main thread.

### 3.2 Security and Privacy

- REQ-SEC-01: App Sandbox shall be disabled to allow full Unix socket and file-system access required for Claude Code integration.
- REQ-SEC-02: Hardened runtime shall be enabled for all release builds.
- REQ-SEC-03: All release binaries shall be notarized with Apple before distribution.
- REQ-SEC-04: Sparkle private keys shall never be committed to version control (stored in `.sparkle-keys/`, gitignored).
- REQ-SEC-05: Analytics shall collect only anonymous usage events (app launches, session starts); no conversation content or personal data shall be transmitted.

### 3.3 Reliability

- REQ-REL-01: The app shall handle Unix socket disconnection and reconnect without requiring a restart.
- REQ-REL-02: A missing or corrupt JSONL file shall not crash the app; the chat view shall show an empty or error state.
- REQ-REL-03: Screen disconnection events shall not crash the app.

### 3.4 Compatibility

- REQ-COMPAT-01: The app shall run on macOS 15.6 and later.
- REQ-COMPAT-02: Both Apple Silicon and Intel architectures shall be supported.
- REQ-COMPAT-03: The app shall function on MacBooks without a notch (non-notch fallback mode).

---

## 4. Integration Requirements

### 4.1 Claude Code CLI

- REQ-INT-01: The app integrates exclusively with the Claude Code CLI via hook scripts in `~/.claude/hooks/`.
- REQ-INT-02: Communication between hooks and the app uses a local Unix domain socket.

### 4.2 Terminal and tmux

- REQ-INT-03: The app shall detect whether the terminal application associated with a session is currently visible on screen.
- REQ-INT-04: The app shall match Claude Code sessions running inside tmux panes to the correct session context.
- REQ-INT-05: The app shall support sending approval responses to the correct tmux pane.

### 4.3 Window Management

- REQ-INT-06: The app shall optionally integrate with yabai for window focus management.
- REQ-INT-07: The app shall locate and focus the terminal window associated with a session when requested.

### 4.4 Analytics

- REQ-INT-08: Anonymous telemetry shall be sent via the Mixpanel SDK.
- REQ-INT-09: Tracked events shall be limited to app lifecycle events (launch, session start).

---

## 5. Build and Release Requirements

### 5.1 Build

- REQ-BUILD-01: The project shall build with Xcode using the Swift Package Manager for dependency resolution.
- REQ-BUILD-02: Release builds shall use the Developer ID distribution certificate and hardened runtime.
- REQ-BUILD-03: The build script (`scripts/build.sh`) shall produce a signed `.app` bundle.

### 5.2 Release Pipeline

- REQ-REL-PIPE-01: Release artifacts shall be notarized using Apple's notarytool before distribution.
- REQ-REL-PIPE-02: A DMG installer shall be generated for each release.
- REQ-REL-PIPE-03: The DMG shall be notarized and signed with the Sparkle EdDSA key.
- REQ-REL-PIPE-04: A GitHub release shall be created automatically via the release script (`scripts/create-release.sh`).
- REQ-REL-PIPE-05: The Sparkle appcast (`appcast.xml`) shall be updated on the website repo for each release.

### 5.3 Testing

- REQ-TEST-01: The project currently has no automated test target; a unit test target shall be added.
- REQ-TEST-02: Critical session state machine transitions (SessionPhase) shall be covered by unit tests.
- REQ-TEST-03: Hook installation and socket communication logic shall be covered by integration tests.

---

## 6. Third-Party Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| Sparkle | Latest | Auto-update framework with EdDSA signing |
| apple/swift-markdown | Latest | Markdown parsing and rendering for chat history |
| mixpanel/mixpanel-swift | Latest | Anonymous usage analytics |
