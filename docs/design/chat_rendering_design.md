---
codd:
  node_id: design:chat-rendering-design
  type: design
  depends_on:
  - id: design:session-lifecycle-design
    relation: depends_on
    semantic: technical
  depended_by:
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:chat-rendering-design
    reason: JSONL parsing must execute off the main thread (REQ-PERF-03). Blocking
      the main thread during file parsing is a release-blocking performance violation.
  - targets:
    - design:chat-rendering-design
    reason: A missing or corrupt JSONL file must not crash the app; the chat view
      must degrade to an empty or error state (REQ-REL-02). Unhandled parse exceptions
      block release.
  - targets:
    - design:chat-rendering-design
    reason: Chat content must never be included in any telemetry payload; only anonymous
      lifecycle events are permitted (REQ-SEC-05).
---

# Chat History and Rendering Design

## 1. Overview

This document specifies the architecture for parsing, caching, and rendering Claude Code session chat history within Claude Island. Chat history is sourced from newline-delimited JSON (JSONL) files written by the Claude Code CLI process into `~/.claude/projects/` and exposed to the overlay via `Session.chatHistoryURL`. The rendering pipeline reads those files asynchronously, caches parsed results in `SessionStore`, and presents them through `ChatHistoryView` — a SwiftUI scroll view that updates incrementally as new messages arrive during a live session.

**Release-blocking constraints owned by this document:**

| ID | Constraint | Enforcement |
|---|---|---|
| REQ-PERF-03 / FC-PERF-03 | JSONL parsing must execute entirely on `ChatActor`; main thread must never block on file I/O or JSON decode | `testMainThreadNotBlockedDuringParse` in `ClaudeIslandTests`; violation blocks release |
| REQ-REL-02 / FC-REL-02 | A missing or corrupt JSONL file must not crash the app; `ChatHistoryView` must render an empty or error state | `testMissingFileProducesEmptyState` and `testCorruptJSONLProducesErrorState` in `ClaudeIslandTests`; unhandled parse exception blocks release |
| REQ-SEC-05 / FC-SEC-02 | Chat message content — including role, text, tool names, tool inputs, and tool results — must never appear in any `AnalyticsClient.track()` call or `MixpanelBridge` payload | Mixpanel call-site audit at release; any content field in a telemetry payload blocks release |

The monitoring latency constraint inherited from `design:session-lifecycle-design` (REQ-PERF-02, ≤ 100 ms socket-to-`@Published`) governs session phase updates but does not govern chat history rendering; chat history is updated on `tool_result` event receipt and is permitted a wider rendering budget (≤ 500 ms file-read-to-rendered, REQ-PERF-04) because it is not on the critical approval-response path.

Claude Code writes JSONL files at paths of the form `~/.claude/projects/<project-hash>/<session-uuid>.jsonl`. Each line is a self-contained JSON object representing one message or tool exchange. The `session_start` hook event includes a `chat_history_url` field pointing to the file for the current session; `SessionStore` stores this as `Session.chatHistoryURL`. If `chat_history_url` is absent from the `session_start` event (for example, when Claude Code is invoked without a project context), `Session.chatHistoryURL` is `nil` and `ChatHistoryView` renders the no-history empty state immediately without attempting any file access.

---

## 2. Architecture

### 2.1 JSONL Message Format

Each line of a Claude Code JSONL history file is a JSON object conforming to one of three message types:

**Human message:**
```json
{"type":"human","uuid":"<uuid>","timestamp":"<iso8601>","content":[{"type":"text","text":"<user input>"}]}
```

**Assistant message:**
```json
{"type":"assistant","uuid":"<uuid>","timestamp":"<iso8601>","content":[{"type":"text","text":"<response text>"},{"type":"tool_use","id":"<tool-use-id>","name":"<tool-name>","input":{}}]}
```

**Tool result:**
```json
{"type":"tool_result","tool_use_id":"<tool-use-id>","timestamp":"<iso8601>","content":[{"type":"text","text":"<output>"}],"is_error":false}
```

Lines that do not parse as valid JSON, lines whose `type` field is unrecognized, and lines whose required fields are missing are individually skipped; they do not terminate parsing of subsequent lines. This per-line fault isolation is the primary mechanism satisfying REQ-REL-02: a single corrupt line degrades one message display, not the entire history view.

### 2.2 Data Model

```swift
enum ChatMessageContent: Equatable {
    case text(String)
    case toolUse(id: String, name: String, inputJSON: String)
    case toolResult(toolUseID: String, outputText: String, isError: Bool)
    case unknown
}

struct ChatMessage: Identifiable, Equatable {
    let id: UUID                        // Parsed from message "uuid" field; generated if absent
    let role: ChatRole                  // .human, .assistant, .toolResult
    let timestamp: Date                 // Parsed from ISO 8601 "timestamp" field
    let contentBlocks: [ChatMessageContent]
}

enum ChatRole: Equatable {
    case human
    case assistant
    case toolResult
}

enum ChatHistoryResult: Equatable {
    case loaded([ChatMessage])
    case empty                          // File absent or zero valid lines
    case error(String)                  // File I/O failure or complete parse failure
}
```

`ChatMessage` is a value type. `contentBlocks` is an immutable array; the data model never mutates in place. `ChatRole.toolResult` is used when `type == "tool_result"` in the JSONL line; tool-use blocks within an assistant message are represented as `ChatMessageContent.toolUse` within the assistant `ChatMessage`.

### 2.3 ChatActor — Background Parsing Actor

`ChatActor` is a Swift global actor that owns all JSONL file I/O and JSON decoding. It is declared:

```swift
@globalActor
actor ChatActor {
    static let shared = ChatActor()
}
```

No parsing method, file handle operation, or `JSONDecoder` call is permitted on the main actor or on an unspecified `Task` without explicit `ChatActor` isolation. This is the architectural guarantee for REQ-PERF-03 (FC-PERF-03). The `testMainThreadNotBlockedDuringParse` test verifies this by running `ChatHistoryParser.parse(url:)` and asserting via `dispatchPrecondition(condition: .notOnQueue(.main))` inside the parser body.

**`ChatHistoryParser`:**

```swift
final class ChatHistoryParser {
    @ChatActor
    func parse(url: URL) async -> ChatHistoryResult {
        // File I/O and per-line JSON decoding runs here, never on MainActor
    }
}
```

Internal implementation:

1. Open the file for reading using `FileHandle(forReadingFrom:)`. If the file does not exist (`ENOENT`) or cannot be opened, return `ChatHistoryResult.empty` — do not throw.
2. Read file content into a `Data` buffer in a single `readDataToEndOfFile()` call. If the file exists but read fails, return `ChatHistoryResult.error("Read failed: \(error.localizedDescription)")`.
3. Split on `\n` using `Data.split(separator: 0x0A)`. Empty lines (length zero) are skipped.
4. For each line, attempt `JSONDecoder().decode(RawJSONLLine.self, from: lineData)`. Lines that throw are counted in `skippedLines: Int` and skipped; they do not propagate.
5. Map each successfully decoded `RawJSONLLine` to a `ChatMessage`. Lines whose `type` is unrecognized produce `ChatMessageContent.unknown` content blocks; they are included in the result (to preserve message ordering) but rendered as a collapsed placeholder in the UI.
6. If the resulting `[ChatMessage]` is empty (all lines skipped or file was empty), return `ChatHistoryResult.empty`.
7. Return `ChatHistoryResult.loaded(messages)`.

The `RawJSONLLine` type is a private intermediate struct used only inside `ChatHistoryParser`. It is not exposed through any public API and does not appear in telemetry or logs — satisfying REQ-SEC-05's requirement that chat content remain entirely within the rendering pipeline.

### 2.4 Cache Architecture in SessionStore

`SessionStore` holds parsed chat history in a dedicated dictionary isolated from the session phase dictionary:

```swift
@MainActor
final class SessionStore: ObservableObject {
    @Published private(set) var sessions: [UUID: Session] = [:]
    @Published private(set) var chatHistory: [UUID: ChatHistoryResult] = [:]
    // ...
}
```

`chatHistory` is keyed by `Session.id` (the session UUID). It is `@Published` so `ChatHistoryView` reacts automatically to updates via the standard SwiftUI observation mechanism.

**Initial load:** On `session_start` event, if `Session.chatHistoryURL` is non-nil, `SessionStore` dispatches a `Task` to `ChatActor` to run `ChatHistoryParser.parse(url:)`. On completion, it dispatches back to `@MainActor`:

```swift
Task {
    let result = await ChatHistoryParser.shared.parse(url: chatHistoryURL)
    await MainActor.run {
        sessionStore.chatHistory[sessionID] = result
    }
}
```

If `Session.chatHistoryURL` is nil, `chatHistory[sessionID]` is set to `.empty` immediately on the main actor without spawning any background task.

**Incremental refresh on tool_result:** Each `tool_result` event may append new assistant and tool-result lines to the JSONL file. On receipt of `tool_result`, `SessionStore.apply(event:)` dispatches a re-parse task to `ChatActor`. The re-parse reads the entire file again (from offset 0), which is acceptable given that Claude Code session JSONL files are bounded by the session's message count and rarely exceed a few hundred kilobytes. Incremental append parsing (reading only new bytes since the last read offset) is reserved as a future optimization once baseline performance is measured.

**Cache invalidation on session end:** When a `session_end` event is received, no additional re-parse is triggered; the last `tool_result`-triggered re-parse is considered the final state. The `chatHistory[sessionID]` entry is retained for the 3-second checkmark display period (matching session retention in `sessions`) and then removed when the session entry is purged.

**No re-parse on session switch:** When `SessionListView` changes `selectedSessionID`, `ChatHistoryView` reads from `chatHistory[newSessionID]` directly. No file I/O is performed on session switch (AC-CHAT-05).

### 2.5 ChatHistoryView — SwiftUI Rendering

`ChatHistoryView` is a `View` that observes `SessionStore` and renders the `ChatHistoryResult` for `selectedSessionID`.

```swift
struct ChatHistoryView: View {
    @EnvironmentObject var store: SessionStore
    let sessionID: UUID

    var body: some View {
        switch store.chatHistory[sessionID] ?? .empty {
        case .loaded(let messages):
            MessageListView(messages: messages)
        case .empty:
            EmptyHistoryView()
        case .error(let description):
            HistoryErrorView(description: description)
        }
    }
}
```

**`MessageListView`:** A `ScrollViewReader`-wrapped `LazyVStack` rendering `MessageRowView` for each `ChatMessage`. New messages are scrolled into view automatically: on each update to `store.chatHistory[sessionID]`, if the result is `.loaded` and the last message ID has changed, `ScrollViewReader.scrollTo(_:anchor: .bottom)` is called with `withAnimation(.easeOut(duration: 0.2))`. Scrolling does not block the main actor; the `scrollTo` call is purely declarative.

**`MessageRowView`:** Renders each `ChatMessage` according to its `ChatRole`:

- `.human` — right-aligned bubble with `Color.accentColor.opacity(0.15)` background; text content rendered with `Text` using `.font(.body)`.
- `.assistant` — left-aligned bubble with `Color.secondary.opacity(0.10)` background; text blocks rendered with `Text`; `toolUse` blocks rendered as a collapsed disclosure group showing `"Tool: <name>"` with a chevron that expands to show the input JSON in a monospaced `TextEditor` (read-only).
- `.toolResult` — left-aligned, reduced-opacity row with a `"→"` prefix; error results (`isError: true`) rendered with `Color.red.opacity(0.15)` background.
- `.unknown` content blocks — rendered as a gray italic placeholder `"[unsupported message type]"` without revealing the raw JSON bytes.

No message text, tool name, tool input, or tool output is passed to `AnalyticsClient` or `MixpanelBridge` at any render path. `MessageRowView` contains no telemetry calls. Compliance with REQ-SEC-05 is enforced structurally: analytics calls exist only in `SessionStore.apply(event:)` for the anonymous `session_started` lifecycle event, which carries no content properties.

**`EmptyHistoryView`:** Displays a centered label `"No history yet"` with `Image(systemName: "clock")`. Used for `ChatHistoryResult.empty` and for sessions with `chatHistoryURL == nil`.

**`HistoryErrorView`:** Displays a centered label `"Could not load chat history"` with `Image(systemName: "exclamationmark.triangle")` and a secondary caption `"The history file may be missing or unreadable."`. The `description` string from `ChatHistoryResult.error` is logged at `os_log` `.error` (subsystem `com.claudeisland.chat`, category `parser`) but is not displayed to the user to avoid leaking internal path or system information. Compliance with REQ-REL-02: this view is the degraded state; the app does not crash, does not present an error dialog, and does not prevent the session phase UI from functioning.

### 2.6 Fault Isolation — REQ-REL-02

Three independent fault boundaries prevent a JSONL issue from crashing the app:

1. **File open failure:** `FileHandle(forReadingFrom:)` failure is caught and produces `ChatHistoryResult.error(...)` before any parsing begins.
2. **Per-line parse failure:** Each line's `JSONDecoder` call is wrapped in a `do/catch`. A thrown error increments `skippedLines` and continues to the next line. The failure of one line cannot affect any other line.
3. **SwiftUI view layer:** `ChatHistoryView` uses a `switch` on `ChatHistoryResult` with an explicit `case .error` branch. No force-unwrap or implicitly unwrapped optional is present in the rendering path. The `?? .empty` fallback for `store.chatHistory[sessionID]` handles the case where no entry exists for the session (e.g., if `session_start` has not yet been processed).

These three boundaries together satisfy FC-REL-02. The unit tests `testMissingFileProducesEmptyState` (passes a nonexistent `URL` to `ChatHistoryParser.parse(url:)`, asserts result equals `.empty`) and `testCorruptJSONLProducesErrorState` (writes a file containing `{"broken": [}` and asserts result is `.empty` or `.error` but not a thrown exception reaching the caller) are required tests; their absence blocks release.

### 2.7 Security and Privacy Controls — REQ-SEC-05

Chat content isolation is enforced at three architectural layers:

**Layer 1 — Data model containment:** `ChatHistoryResult` and `ChatMessage` values are stored exclusively in `SessionStore.chatHistory`. No other system component holds a reference to parsed chat content. `PermissionRouter`, `SocketActor`, `SoundPlayer`, and `NotchWindow` have no access to `chatHistory`.

**Layer 2 — Analytics call-site discipline:** `AnalyticsClient.track()` is called in exactly two places in the session monitoring subsystem: `SessionStore.apply(event:)` on `session_start` (emitting the `session_started` event with zero properties) and `SessionStore.apply(event:)` on `session_end` (emitting the `session_ended` event with zero properties). No call to `AnalyticsClient.track()` exists in `ChatHistoryParser`, `ChatHistoryView`, `MessageListView`, `MessageRowView`, or any type that holds a `ChatMessage` value. The release-time Mixpanel call-site audit (AC-SEC-03, FC-SEC-02) statically verifies this by grepping the compiled source for `AnalyticsClient.track` call sites and asserting none appear in files under `Chat/`.

**Layer 3 — Logging discipline:** `os_log` calls within `ChatHistoryParser` log file paths and line counts only (`"Parsed \(messageCount) messages from \(url.lastPathComponent)"`). Message content, role strings, tool names, and tool inputs are not included in any log message. The `HistoryErrorView` displays a generic error string, not the file path or error description.

**App Sandbox:** As specified in `design:session-lifecycle-design` §2.11 and ADR-001, the App Sandbox entitlement is absent. `ChatHistoryParser` accesses `~/.claude/projects/` via standard `FileHandle` POSIX calls. No additional entitlements beyond those required for the session monitoring subsystem are needed for chat history file access.

### 2.8 Performance Budget — Chat History Rendering

| Stage | Budget | Notes |
|---|---|---|
| `ChatActor` file read | ≤ 200 ms | For files up to 1 MB; typical session files < 100 KB |
| Per-line JSON decode | ≤ 100 ms | For up to 500 messages; typically < 20 ms |
| `@MainActor` dictionary update + `@Published` | ≤ 30 ms | O(1) dictionary update |
| `LazyVStack` SwiftUI diff + render | ≤ 170 ms | Lazy rendering; only visible rows materialized |
| **Total (file-read-to-rendered)** | **≤ 500 ms** | REQ-PERF-04 |

The 500 ms budget (REQ-PERF-04) applies to the initial history load on `session_start`. Incremental updates on `tool_result` are expected to complete within the same budget because the file size grows incrementally and the `LazyVStack` diff is small (one or two new rows).

The ≥ 60 fps animation requirement (REQ-PERF-01, FC-PERF-02) applies to `ChatHistoryView` scroll animations. `LazyVStack` ensures off-screen rows are not rendered; `MessageRowView` performs no synchronous I/O or heavy computation during layout.

### 2.9 Unit and Integration Test Requirements

| Test method | What it verifies | Failure criterion if absent |
|---|---|---|
| `testMissingFileProducesEmptyState` | `parse(url:)` with nonexistent URL returns `.empty`, no crash | FC-REL-02 |
| `testCorruptJSONLProducesErrorState` | Corrupt file returns `.empty` or `.error`, no thrown exception | FC-REL-02 |
| `testMainThreadNotBlockedDuringParse` | `dispatchPrecondition(.notOnQueue(.main))` passes inside parser | FC-PERF-03 |
| `testPartiallyCorruptJSONLSkipsBadLines` | File with 3 valid lines + 1 corrupt line returns `.loaded` with 3 messages | Regression guard for per-line fault isolation |
| `testChatHistoryUpdatesOnToolResult` | `tool_result` event triggers re-parse; `chatHistory[sessionID]` updates | Regression guard for incremental refresh |
| `testNoSessionSwitchTriggersReparse` | Changing `selectedSessionID` does not call `ChatHistoryParser.parse` | AC-CHAT-05 |
| `testNilChatHistoryURLProducesEmpty` | Session with `chatHistoryURL == nil` immediately yields `.empty` without file access | Guard against spurious I/O |
| `testAnalyticsContainsNoChatContent` | Swizzled `AnalyticsClient.track` asserts zero properties in all calls during a full parse + render cycle | FC-SEC-02 |

---

## 3. Open Questions

**OQ-CHR-01 — Incremental Append Parsing vs. Full Re-read on tool_result**
The current design re-reads the entire JSONL file on each `tool_result` event. For long sessions with hundreds of messages, repeated full re-reads accumulate unnecessary I/O and decode work. An incremental approach — tracking the byte offset of the last read and seeking to that position before reading new lines — would reduce per-update work to O(new lines). However, incremental parsing requires that `ChatHistoryParser` retain per-session read-offset state, introducing statefulness that complicates testing. The chosen approach (full re-read) is correct and satisfies REQ-PERF-04 for typical session sizes. A file-size threshold above which incremental parsing activates (e.g., > 500 KB) must be defined before GA if profiling reveals a regression on very long sessions.

**OQ-CHR-02 — Rendering of Binary or Image Tool Results**
Claude Code tool results may include base64-encoded image data (e.g., from screen capture tools). The current `ChatMessageContent.toolResult` model stores only `outputText: String`. A tool result whose content block is `{"type":"image","source":{"type":"base64","media_type":"image/png","data":"..."}}` would be stored as an empty `outputText` or as a raw base64 string, neither of which renders usefully. A `ChatMessageContent.image(data: Data, mediaType: String)` case and a corresponding `ImageMessageView` using `Image(nsImage: NSImage(data:))` must be specified before rendering is considered complete for tool-heavy sessions. This is deferred to a post-1.0 iteration unless image-producing tools are identified as common in the initial user population.

**OQ-CHR-03 — Maximum Displayed Message Count and Virtualization**
`LazyVStack` defers off-screen row creation but does not limit the total number of `ChatMessage` values held in memory for a single session. For sessions with thousands of messages (long agentic runs), the `[ChatMessage]` array may consume significant memory. A cap on the displayed window (e.g., most recent 200 messages, with a "load earlier messages" control) or a true windowed virtualization approach using `List` with `.listStyle(.plain)` must be evaluated. The memory and scroll-performance impact of unbounded message arrays must be measured on hardware representative of the target minimum spec (M1 MacBook Air) before 1.0 GA. Until that measurement is complete, the architecture should be considered provisional for sessions exceeding 500 messages.

**OQ-CHR-04 — Handling of Simultaneous JSONL File Writes During Parse**
Claude Code writes to the JSONL file while a session is active. If `ChatHistoryParser` is in the middle of a `readDataToEndOfFile()` call at the same moment Claude Code appends a new line, the read may capture a partial line (a line whose terminal `\n` byte has not yet been written). The per-line `JSONDecoder` call will fail for the partial line and skip it. On the next re-parse (triggered by the following `tool_result` event), the line will be complete and will parse successfully. This is the intended behavior; the question is whether this transient gap (one message absent for the duration between two `tool_result` events) is acceptable to users or whether it requires a smarter end-of-file truncation heuristic. A formal decision must be recorded in the detailed design before the chat rendering subsystem ships.

**OQ-CHR-05 — Cross-Session chat_history_url Collision Handling**
`Session.chatHistoryURL` is populated from the `session_start` event's `chat_history_url` field. If two simultaneous sessions point to the same JSONL file path (which can happen if Claude Code is invoked twice in the same project directory without distinct session UUIDs), `SessionStore.chatHistory` would store the same `ChatHistoryResult` under two different `UUID` keys. Updates triggered by one session's `tool_result` events would also appear in the other session's `ChatHistoryView`. The probability of this scenario depends on Claude Code's JSONL file-naming strategy. If Claude Code guarantees per-session unique file paths (as implied by the `<session-uuid>.jsonl` naming convention), this is a non-issue. The Claude Code JSONL file path guarantees must be confirmed and documented in the IPC hook specification before this concern is formally closed.
