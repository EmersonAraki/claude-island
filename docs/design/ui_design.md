---
codd:
  node_id: design:ui-design
  type: design
  depends_on:
  - id: design:system-design
    relation: depends_on
    semantic: technical
  depended_by:
  - id: detail:notch_overlay_state_machine
    relation: depends_on
    semantic: technical
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:ui-design
    reason: Overlay must animate at 60 fps using spring animations with no visible
      jank (REQ-PERF-01, REQ-UI-03). Frame-drop risk must be assessed for all state
      transitions.
  - targets:
    - design:ui-design
    reason: Non-notch fallback to a floating pill at the top of the selected screen
      is mandatory for release on MacBooks without a notch (REQ-UI-04, REQ-COMPAT-03).
  - targets:
    - design:ui-design
    reason: Screen selection must persist across restarts using a stable display identifier
      (REQ-SCREEN-03). Graceful fallback to an available screen on disconnect is required
      and must not crash the app (REQ-SCREEN-04, REQ-REL-03).
---

# Notch Overlay UI Design

## 1. Overview

The Notch Overlay is the primary visual surface of Claude Island. It renders as a borderless, non-activating `NSWindow` anchored to the notch region on notch-equipped MacBooks. On hardware without a notch — including Mac mini, non-notch MacBook Pro models, and any external display — it falls back to a floating pill positioned at the top center of the selected screen. This non-notch fallback is mandatory for release; its absence is failure criterion FC-UI-01 (REQ-UI-04, REQ-COMPAT-03).

The overlay surfaces four categories of content without requiring the user to switch application windows:

- **Session status** — phase indicator (idle, processing, waiting for input, ended), elapsed time, and animated crab icon during active processing.
- **Tool permission requests** — tool name, complete tool input, and approve/deny controls, delivered within 100 ms of socket message receipt (AC-PERM-01).
- **Chat history** — Markdown-rendered message thread using `apple/swift-markdown`, auto-scrolling to the latest message with a manual-scroll pause affordance.
- **Response input** — interactive prompt bar that appears when the `AskUserQuestion` tool is active (`waitingForInput` phase).

All state transitions animate at ≥ 60 fps using SwiftUI spring animations (REQ-PERF-01). Frame-drop risk is assessed across every transition listed in §2.2; each transition is instrumented in CI via Instruments Time Profiler. A sub-60-fps result on any transition constitutes FC-PERF-02, a release-blocking failure.

Screen selection persists across restarts using a stable `NSScreen` display identifier stored in `UserDefaults` (REQ-SCREEN-03). If the persisted screen is disconnected at launch or at runtime, `ScreenSelector` falls back silently to the first entry in `NSScreen.screens` without crashing (REQ-SCREEN-04, AC-SCREEN-04, FC-REL-03).

---

## 2. Architecture

### 2.1 Window Configuration

The overlay is an `NSWindow` subclass with the following properties set at initialization:

| Property | Value | Rationale |
|---|---|---|
| `styleMask` | `.borderless` | No title bar or chrome |
| `level` | `NSWindow.Level.statusBar + 1` | Floats above ordinary windows and full-screen apps |
| `collectionBehavior` | `.canJoinAllSpaces \| .stationary \| .ignoresCycle` | Visible on all Spaces; excluded from Cmd-Tab cycling |
| `isOpaque` | `false` | Required for rounded-pill shape with transparent background |
| `hasShadow` | `true` | Depth cue against light desktop backgrounds |
| `ignoresMouseEvents` | `false` (only while expanded) | Tap-to-open; mouse passes through when closed |
| `canBecomeKey` | `false` | Does not steal keyboard focus from the terminal |

`NSApp.activate` is never called in response to overlay interactions. The terminal retains keyboard focus at all times unless the user explicitly taps the input bar in `ChatHistoryView` during a `waitingForInput` phase.

### 2.2 Notch Anchor vs. Floating-Pill Fallback

At app launch, `ScreenSelector` evaluates the selected screen. Notch presence is determined by checking whether `NSScreen.safeAreaInsets.top > 0` on the target display. This check is re-evaluated whenever `NSApplication.didChangeScreenParametersNotification` fires.

**Notch mode (REQ-UI-01):** The window frame is calculated so the pill shape fills the notch bounding rectangle exactly. Width is fixed to the notch width; height extends 4 pt below the notch bottom edge to allow the spring-expand animation to reveal content downward.

**Floating-pill fallback (REQ-UI-04, REQ-COMPAT-03):** When `NSScreen.safeAreaInsets.top == 0`, the pill is centered horizontally at the top of the selected screen with a configurable top margin of 8 pt from the menu bar bottom edge. Width and height match the notch-mode dimensions so appearance is consistent across hardware. This fallback path is exercised in CI on a non-notch target.

Both modes share the same `NotchOverlayView` SwiftUI content. The `NSHostingView` frame is the only variable.

### 2.3 State Machine and Transitions

`NotchOverlayView` is driven entirely by `SessionStore`, an `ObservableObject` on the main actor. The view observes `@Published var sessions: [UUID: Session]` and `@Published var activeSessionID: UUID?`. All state transitions use SwiftUI spring animations to satisfy REQ-PERF-01 (≥ 60 fps).

**Overlay visibility states:**

```
closed ──tap notch / permission arrives──▶ popping
popping ──spring animation complete──▶ opened
opened ──dismiss gesture──▶ closing
closing ──spring animation complete──▶ closed
opened ──session ends (success)──▶ opened(green, 3 s) ──timer──▶ closing
opened ──permission arrives──▶ opened(expanded, amber)
```

**Frame-drop risk assessment per transition:**

| Transition | Animated properties | Risk | Mitigation |
|---|---|---|---|
| `closed → popping` | frame height, opacity, blur radius | Medium — simultaneous geometry + blur | Animate blur with `.animation(.spring, value:)` separately from geometry; pre-warm layer |
| `popping → opened` | frame height spring settle | Low | Single geometry animation |
| `opened → closing` | frame height, opacity | Low | Opacity on separate layer |
| `opened (expanded)` | frame height for permission card | Medium — content layout change | Wrap content in `matchedGeometryEffect`-free layout; measure with fixed frames |
| `opened(green) → closing` | color transition + frame collapse | Low | Color via `.foregroundStyle` not layer background |
| `waitingForInput` bounce | `offset` Y pulse | Low | Use `.spring(response:dampingFraction:)` with `repeatCount(1, autoreverses: true)` |

CI gate: Instruments Time Profiler `xcrun xctrace` run on a Release build on both Apple Silicon and Intel targets. Any frame exceeding 16.67 ms (60 fps budget) during the above transitions fails FC-PERF-02 and blocks release.

### 2.4 View Hierarchy

```
NotchOverlayWindow (NSWindow subclass)
└── NSHostingView<NotchOverlayView>
    └── NotchOverlayView
        ├── PillShapeBackground          (GeometryReader, rounded rect, blur material)
        ├── CompactBar                   (always visible when opened)
        │   ├── CrabIconView             (animated during processing phase)
        │   ├── SessionTitleLabel
        │   ├── ElapsedTimeLabel
        │   └── PhaseIndicatorDot        (grey=idle, amber=waitingForInput, green=ended/success)
        ├── ExpandedContent              (shown when opened and expanded)
        │   ├── ChatHistoryView
        │   │   ├── MessageList          (LazyVStack, auto-scroll)
        │   │   │   ├── AssistantMessageView   (swift-markdown rendered)
        │   │   │   ├── UserMessageView
        │   │   │   └── ToolCallView     (name + args + status badge)
        │   │   ├── ScrollToBottomButton (appears on manual scroll-up)
        │   │   └── AskInputBar          (visible in waitingForInput phase only)
        │   └── PermissionRequestView    (overlays ChatHistoryView when pendingPermission ≠ nil)
        │       ├── ToolNameLabel
        │       ├── ToolInputScrollView
        │       ├── ApproveButton
        │       └── DenyButton
        └── SettingsButton               (gear icon, opens SettingsView as sheet)
```

### 2.5 CrabIconView and Phase Indicators

`CrabIconView` renders the animated crab asset. Animation state is gated on `SessionPhase`:

- `processing` → animation plays continuously (Lottie-style frame-by-frame or SwiftUI `TimelineView` at 24 fps asset frame rate).
- `idle` or `ended` → animation halts on the first frame; no CPU is consumed.
- `waitingForInput` → animation halts; amber dot appears on `PhaseIndicatorDot` (AC-UI-06).

The `PhaseIndicatorDot` uses a 6 pt circle. Color mapping:

| Phase | Color | Accessibility label |
|---|---|---|
| `idle` | systemGray | "Idle" |
| `processing` | systemBlue | "Processing" |
| `waitingForInput` | systemOrange | "Waiting for input" |
| `ended` (success) | systemGreen | "Complete" |
| `ended` (error) | systemRed | "Error" |

### 2.6 Overlay Bounce (AC-UI-08)

When `SessionPhase` transitions to `waitingForInput` and the terminal window hosting the session does not have keyboard focus (detected via `NSWorkspace.shared.frontmostApplication`), the overlay emits a vertical bounce animation:

```swift
withAnimation(.spring(response: 0.3, dampingFraction: 0.4).repeatCount(1, autoreverses: true)) {
    bounceOffset = -8
}
```

The bounce is a single-cycle spring, not a looping animation. It fires once on phase entry and does not repeat unless the phase exits and re-enters `waitingForInput`.

### 2.7 Permission Request View (AC-PERM-01 – AC-PERM-05)

When `SessionStore` sets `session.pendingPermission` to a non-nil `PermissionRequest`, `PermissionRequestView` overlays the expanded content area. Layout:

1. **Tool name** — bold system font, left-aligned.
2. **Tool input scroll view** — monospaced font, max height 180 pt, vertical scroll if content exceeds. The complete tool input is always shown; no truncation (AC-PERM-01).
3. **Approve / Deny buttons** — trailing-aligned HStack. Approve uses `buttonStyle(.borderedProminent)` with green tint; Deny uses `.bordered` with default tint.

Tapping Approve or Deny calls `PermissionRouter.respond(sessionID:approved:)`, which writes the approval/denial JSON back to the exact `UnixSocketServer` client connection that originated the request (AC-PERM-03). For sessions with a non-nil `tmuxPane`, `PermissionRouter` additionally delivers the response to the identified tmux pane (AC-PERM-05). Routing must be verified with two concurrent Claude Code sessions in separate tmux panes; misrouting constitutes FC-PERM-01 (release-blocking).

If the terminal is not visible when the permission request arrives, the overlay opens automatically by setting `overlayState = .opened` before the spring animation fires (AC-PERM-04).

The entire path from socket receipt to `PermissionRequestView` render must complete within 100 ms (AC-PERM-01), consistent with the socket-to-UI latency requirement (REQ-PERF-02, FC-PERF-01).

### 2.8 Chat History View (AC-CHAT-01 – AC-CHAT-06)

`ChatHistoryView` presents `session.chatHistory`, a `[ChatMessage]` array populated by `ChatHistoryParser`. Parsing runs on a background actor (`ChatParseActor`); results are published to the main actor via `@MainActor func updateHistory(_ messages: [ChatMessage])`. JSONL parsing must not block the main thread (REQ-PERF-03, FC-PERF-03).

**Markdown rendering (AC-CHAT-02):** `AssistantMessageView` passes message body text through `apple/swift-markdown`'s `Document` parser and renders the resulting tree into a SwiftUI `Text` or `AttributedString`. Supported elements: bold, italic, inline code, fenced code blocks, unordered lists, ordered lists. Unknown elements degrade to plain text.

**Tool call inline rendering (AC-CHAT-04):** `ToolCallView` shows:
- Tool name (bold)
- Arguments formatted as pretty-printed JSON (monospaced, max 4 visible lines, expandable)
- Status badge: "pending" (grey), "success" (green), "failure" (red)

**Auto-scroll (AC-CHAT-03):** `MessageList` uses a `ScrollViewReader`. On each new message append, `scrollTo(latestID, anchor: .bottom)` fires with `.animation(.easeOut(duration: 0.2))` unless `isUserScrolling == true`. `isUserScrolling` is set to `true` on `DragGesture` detection in the scroll view and reset to `false` when `ScrollToBottomButton` is tapped or when the session phase changes (new message from a different phase clears the manual-scroll override). `ScrollToBottomButton` appears when scroll position is more than 40 pt above the bottom.

**Session switching without flicker (AC-CHAT-05):** `SessionStore` caches `[UUID: [ChatMessage]]`. Switching `activeSessionID` reads from cache synchronously on the main actor; no re-parse occurs. Cache is invalidated only when the JSONL file modification timestamp changes.

**AskInputBar (AC-CHAT-06, AC-ASK-01, AC-ASK-02):** Visible only in `waitingForInput` phase. Contains:
- `TextField` with placeholder "Type your response…" — focused automatically via `.focused($inputFocused)` when phase enters `waitingForInput`.
- Submit button (return key or tap). On submit, `module:ipc` writes the response string as a JSON object `{"type":"ask_response","text":"<user text>"}` to the originating socket connection.

### 2.9 Settings Integration

The gear button at the bottom-right of the expanded overlay opens `SettingsView` as a SwiftUI sheet anchored to the overlay window — not as a separate window, and not via the menu bar icon (AC-SET-01). The sheet contains exactly six items in the following order (AC-SET-02):

1. **Screen picker** — `Picker` populated from `ScreenSelector.availableScreens`. Selection persists via `UserDefaults` key `selectedDisplayIdentifier` using the stable `CGDirectDisplayID`-based identifier (REQ-SCREEN-03).
2. **Inline scrollable sound picker** — `ScrollView(.horizontal)` containing ≥ 14 system sound names (AC-SOUND-02, AC-SOUND-03). Selection persists in `UserDefaults` (AC-SOUND-04).
3. **Launch-at-login toggle** — calls `SMAppService.mainApp.register()` / `.unregister()` (AC-SET-03).
4. **Hooks installer toggle** — calls `HookInstaller.enable()` / `.disable()`. Enabling requires an explicit confirmation alert before any files are written to `~/.claude/hooks/` (AC-HOOK-01, FC-HOOK-01 — release-blocking: no files may be written without user consent). Toggle state persists in `UserDefaults` (AC-HOOK-03).
5. **Accessibility permissions status** — reads `AXIsProcessTrusted()`. Displays "Granted" in green or "Not Granted" in orange with a "Open System Settings" button that calls `NSWorkspace.shared.open(privacyURL)` (AC-SET-04).
6. **Auto-update control** — Sparkle `SPUStandardUpdaterController` bound toggle for "Automatically check for updates", plus "Check Now" button. Green dot badge on the gear button in `CompactBar` when `updaterDelegate.updateIsAvailable == true` (AC-UPD-03).

### 2.10 Sound Notification Integration

`SoundPlayer` (wrapping `AVFoundation`) fires when a session transitions to `ended` with a success result (AC-SOUND-01). Before playing, `SoundPlayer` checks whether the terminal application hosting the completing session is the frontmost application via `NSWorkspace.shared.frontmostApplication.bundleIdentifier`. If so, no sound plays (AC-SOUND-05). The selected sound name is read from `UserDefaults` key `completionSoundName`; default is `"Glass"`.

### 2.11 Multi-Monitor and Screen Persistence

`ScreenSelector` publishes `@Published var selectedScreen: NSScreen`. On init:

1. Read `UserDefaults["selectedDisplayIdentifier"]`.
2. Find a matching screen in `NSScreen.screens` by comparing `CGDirectDisplayID` values.
3. If no match (screen disconnected), use `NSScreen.screens.first` (AC-SCREEN-04, FC-REL-03 — must not crash).
4. If `UserDefaults` has no value, default to `NSScreen.main ?? NSScreen.screens.first` (AC-SCREEN-02).

When `NSApplication.didChangeScreenParametersNotification` fires, `ScreenSelector` re-evaluates. If the selected screen has been removed from `NSScreen.screens`, fallback applies automatically. The overlay window frame is updated via `NotchOverlayWindow.updateFrame(for: selectedScreen)` without closing or recreating the window.

### 2.12 Performance Compliance

The following measurable thresholds apply to the overlay UI and are enforced as release-blocking gates:

| Requirement ID | Threshold | Failure Criterion | Measurement Method |
|---|---|---|---|
| REQ-PERF-01 | ≥ 60 fps on all state transitions | FC-PERF-02 | `xcrun xctrace` Time Profiler; frame budget 16.67 ms |
| REQ-PERF-02 | Socket-to-UI latency ≤ 100 ms (±10 ms) | FC-PERF-01 | Timestamp at `SocketActor` frame receipt and at `@Published` update on main actor |
| REQ-PERF-03 | JSONL parse must not block main thread | FC-PERF-03 | `ChatParseActor` background actor; assert `Thread.isMainThread == false` inside parser |
| REQ-UI-03 | Spring animations, no visible jank | REQ-PERF-01 / FC-PERF-02 | Same CI gate as REQ-PERF-01 |

### 2.13 Security and Access Control Constraints

The overlay UI complies with the following release-blocking security constraints from the system design:

- **App Sandbox disabled (REQ-SEC-01):** The overlay window requires unrestricted `NSScreen` and `NSWindow` access for level and collection-behavior configuration. CI asserts `com.apple.security.app-sandbox = false` in the entitlements file before code-signing.
- **Hardened runtime (REQ-SEC-02):** `ENABLE_HARDENED_RUNTIME = YES` is set; no JIT or dyld-environment-variable entitlements are introduced by the overlay implementation.
- **No focus steal:** The overlay never calls `NSApp.activate` or `window.makeKey`. Keyboard focus remains with the terminal, preventing any credential-entry interception risk.
- **Hook consent gate (AC-HOOK-01, FC-HOOK-01):** `SettingsView` wraps the hooks installer toggle in an `Alert` confirmation before calling `HookInstaller.enable()`. The button action is the sole code path that can trigger file writes to `~/.claude/hooks/`; there is no background or automatic write path.
- **Anonymous telemetry (REQ-SEC-05):** `NotchOverlayView`, `ChatHistoryView`, `PermissionRequestView`, and `AskInputBar` contain no import of `mixpanel-swift` and make no direct calls to `MixpanelBridge`. All telemetry flows exclusively through `AnalyticsClient`. The two permitted events (`app_launched`, `session_started`) carry no message content, tool names, file paths, or user-identifying data. Compliance is verified at release by auditing every `AnalyticsClient` call site (AC-SEC-03, FC-SEC-02 — release-blocking).
- **Notarization (REQ-SEC-03):** The overlay is part of the notarized `.app` bundle. `xcrun notarytool submit --wait` and `xcrun stapler validate` must both succeed before the DMG is published (FC-SEC-03).

### 2.14 Reliability Constraints

- **FC-REL-03 — Display disconnect:** `ScreenSelector` fallback path has a dedicated unit test asserting that removing the selected screen from a mock `NSScreen.screens` array results in a non-nil `selectedScreen` pointed at the fallback and does not throw.
- **FC-REL-02 — Corrupt JSONL:** `ChatHistoryParser` wraps all file I/O in a `do/catch`. A missing, empty, or malformed file produces `Result.failure(ChatParseError.unreadable)`, which `ChatHistoryView` renders as an empty message list with a localised error label. No crash path exists.
- **FC-REL-01 — Socket reconnection:** While this is owned by `module:ipc`, the overlay's `SessionStore` observer must tolerate sessions disappearing and reappearing without resetting manually-scrolled positions or losing the selected `activeSessionID` unless the session is permanently removed.

---

## 3. Open Questions

**OQ-UI-001 — Notch Width Variability Across Models**
MacBook Pro notch widths differ between the 14-inch and 16-inch models and may differ on future hardware. The current design reads `NSScreen.safeAreaInsets` to derive the notch bounding rectangle, but the exact pixel width and whether `safeAreaInsets.top` provides sufficient precision for sub-point alignment has not been verified on all shipping notch configurations. This must be validated on physical hardware for each supported notch geometry before release.

**OQ-UI-002 — Input Bar Focus and Keyboard Routing**
When `AskInputBar` is focused via `.focused($inputFocused)` in `waitingForInput` phase, key events are routed to the overlay window's `NSHostingView`. The interaction between `canBecomeKey = false` on the window and the `@FocusState` binding is untested on macOS 15.6. If `canBecomeKey` must be temporarily set to `true` to accept text input, the keyboard-focus-steal risk described in §2.1 applies and must be re-evaluated. A concrete solution — either permitting key focus for the duration of input only or using a separate input window — must be decided before the `AskUserQuestion` feature ships.

**OQ-UI-003 — tmux Pane Routing Protocol (inherits OQ-007 from system design)**
`PermissionRequestView` displays `Approve` and `Deny` for a specific session. If the `tmuxPane` field in the `Session` model is nil at approval time due to the unresolved routing protocol (system design OQ-007), the approval response cannot be delivered to the correct pane. The UI must either block Approve/Deny when `tmuxPane` is nil for a tmux session, or fall back to socket-only delivery with a visible warning. The chosen behavior must be specified and covered by an integration test before the hooks feature ships (FC-PERM-01 is release-blocking).

**OQ-UI-004 — Telemetry Opt-Out UI Placement (inherits OQ-001 from system design)**
If EU distribution requires a `UserDefaults`-gated opt-out for anonymous telemetry (system design OQ-001), the Settings panel must include a seventh item: an "Allow anonymous analytics" toggle. This would exceed the AC-SET-02 count of exactly six items. It is unclear whether AC-SET-02 should be updated to permit seven items, or whether the opt-out should appear only in a secondary "Advanced" section. This must be resolved before EU distribution begins.

**OQ-UI-005 — Accessibility (VoiceOver) for Overlay Window Level**
`NSWindow.Level.statusBar + 1` windows are not standard document windows and their VoiceOver traversal behavior is unspecified in Apple documentation. Whether `AccessibilityElement` annotations on `NotchOverlayView` are sufficient for VoiceOver users to operate Approve/Deny and the AskInputBar, or whether additional `NSAccessibility` protocol conformance on the `NSWindow` subclass is required, has not been evaluated. An accessibility audit against WCAG 2.1 AA is recommended before public distribution but has not been scheduled.
