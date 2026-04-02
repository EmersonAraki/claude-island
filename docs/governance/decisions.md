---
codd:
  node_id: governance:decisions
  type: governance
  depends_on:
  - id: req:claude-island-requirements
    relation: derives_from
    semantic: governance
  depended_by:
  - id: design:system-design
    relation: constrained_by
    semantic: governance
  - id: design:release-pipeline-design
    relation: constrained_by
    semantic: governance
  conventions:
  - targets:
    - governance:decisions
    reason: App Sandbox must be disabled for Unix socket and filesystem access (REQ-SEC-01);
      hardened runtime must be enabled on all release builds (REQ-SEC-02). Both decisions
      are release-blocking and must be formally recorded.
  - targets:
    - governance:decisions
    reason: Sparkle EdDSA private keys must never be committed to version control
      (REQ-SEC-04). This constraint must be documented as an ADR enforced in CI and
      the release pipeline.
  - targets:
    - governance:decisions
    reason: All release binaries must be notarized with Apple before distribution
      (REQ-SEC-03, REQ-REL-PIPE-01). Shipping without notarization is release-blocking.
---

# Architecture Decision Records

## 1. Overview

This document records the binding architectural decisions made for Claude Island, a native macOS menu bar application that renders a Dynamic Island-style notch overlay for Claude Code CLI session monitoring. Each record captures the decision context, the chosen option, the rationale, and the consequences. Decisions marked **RELEASE-BLOCKING** must be satisfied before any distribution artifact ships; violating them halts the release pipeline.

The records in this document are authoritative. Where a decision conflicts with a default framework behavior or convention, this document takes precedence. All decisions reflect requirements from `req:claude-island-requirements`.

---

## 2. Decision Log

### ADR-001 — App Sandbox Disabled for Runtime Entitlements

**Status:** Accepted — RELEASE-BLOCKING (REQ-SEC-01)

**Date:** 2026-04-01

**Context:**
Claude Island must monitor Unix domain socket connections established by hook scripts in `~/.claude/hooks/`, read JSONL chat history files from the user's home directory, detect and communicate with running Claude Code CLI processes, and optionally interact with tmux panes and terminal windows. Apple's App Sandbox restricts all of these capabilities: outbound Unix socket connections outside the app container are prohibited, arbitrary filesystem reads under `~/.claude/` are not permitted without a powerbox-based open panel, and process inspection APIs are blocked.

Enabling the App Sandbox with the required entitlements (`com.apple.security.temporary-exception.mach-lookup.global-name`, custom socket path exceptions) does not provide a supported path for the full set of access patterns required. Attempting to sandbox the app would require a proliferation of hardcoded temporary exceptions that Apple may reject at notarization time and would break on OS upgrades.

**Decision:**
App Sandbox (`com.apple.security.app-sandbox`) is permanently disabled in the `ClaudeIsland.entitlements` file for all build configurations (Debug, Release, Distribution). This is a release-blocking constraint: any build with App Sandbox enabled must not be distributed.

**Compliance control:**
- The entitlements file must not contain `<key>com.apple.security.app-sandbox</key><true/>`.
- CI validates the entitlements file on every build; a CI job step asserts the key is absent or set to `false` before proceeding to code signing.
- Release checklist item: confirm `codesign -d --entitlements - ClaudeIsland.app` does not include the sandbox entitlement.

**Consequences:**
- The app runs outside the App Sandbox, giving it unrestricted filesystem and socket access.
- Distribution via the Mac App Store is not possible; the app is distributed as a signed DMG with Developer ID.
- The absence of sandboxing increases the importance of all other security controls (hardened runtime, notarization, principle of least privilege in code).
- Users must explicitly grant Accessibility permissions (tracked in REQ-SET-02, REQ-SET-04) because App Sandbox cannot provide the entitlement path; the app guides users through this via the Settings panel.

---

### ADR-002 — Hardened Runtime Enabled on All Release Builds

**Status:** Accepted — RELEASE-BLOCKING (REQ-SEC-02)

**Date:** 2026-04-01

**Context:**
With App Sandbox disabled (ADR-001), hardened runtime is the primary security boundary preventing dynamic code injection, use of deprecated APIs that expose attack surface, and loading of unsigned dynamic libraries at runtime. Hardened runtime (`com.apple.security.cs.allow-unsigned-executable-memory` absent, `com.apple.security.cs.disable-library-validation` absent) is also a prerequisite for Apple notarization (ADR-004). Without it, the binary can be notarized only with explicit justification and Apple may reject the submission.

**Decision:**
Hardened runtime is enabled on all Release and Distribution build configurations via the Xcode build setting `ENABLE_HARDENED_RUNTIME = YES`. Debug builds may optionally disable it to allow debugger attachment, but the build script `scripts/build.sh` must always produce artifacts with hardened runtime active.

No hardened-runtime exception entitlements (`cs.allow-jit`, `cs.disable-library-validation`, `cs.allow-dyld-environment-variables`) shall be added without a documented follow-up ADR explaining the necessity and mitigating controls.

**Compliance control:**
- `scripts/build.sh` passes `-hardened-runtime` to `codesign` for the final signing step.
- CI asserts `codesign -dv ClaudeIsland.app 2>&1 | grep "flags=0x" | grep -v "0x0"` confirms the hardened-runtime flag (bit `0x10000`) is set.
- Release checklist item: `spctl -a -vvv ClaudeIsland.app` must return `source=Developer ID`.

**Consequences:**
- Sparkle's DSA/EdDSA update installation requires hardened runtime; this is compatible with Sparkle's XPC-based installer.
- The Mixpanel SDK and apple/swift-markdown must not require JIT entitlements or disable library validation; this is verified at dependency update time.
- Debugging in Xcode is unaffected because Xcode injects the `get-task-allow` entitlement into Debug builds automatically.

---

### ADR-003 — Sparkle EdDSA Keys Excluded from Version Control

**Status:** Accepted — RELEASE-BLOCKING (REQ-SEC-04)

**Date:** 2026-04-01

**Context:**
Claude Island distributes over-the-air updates via the Sparkle framework. Sparkle verifies update packages using an EdDSA (Ed25519) signature; the private key signs the `.delta` or `.zip` archive and the signature is embedded in `appcast.xml`. If the private key is committed to version control, any party with repository read access — including future public forks, CI log leaks, or GitHub token compromise — can produce signed update packages that Sparkle will accept as authentic. A compromised update key enables arbitrary code execution on all installed instances of the app.

**Decision:**
The Sparkle EdDSA private key is stored exclusively in `.sparkle-keys/` on the release engineer's local machine and in a secrets manager (never in the repository). The `.sparkle-keys/` directory is listed in `.gitignore` and in `.git/info/exclude`. The public key is embedded in `Info.plist` under `SUPublicEDKey` and is safe to commit.

A pre-commit hook asserts that no file matching `*.key`, `*.pem`, `*.p8`, `private_key`, or `sparkle_private*` is staged. CI runs a secret-scanning step (using `trufflehog` or equivalent) on every pull request and blocks merge if a private key pattern is detected.

Key rotation procedure: generate a new Ed25519 keypair with `generate_keys` (Sparkle CLI tool), update `SUPublicEDKey` in `Info.plist`, re-sign all appcast entries with `sign_update`, and revoke the old key from the secrets manager. A new ADR addendum records the rotation date and reason.

**Compliance control:**
- `.gitignore` entry: `.sparkle-keys/`
- CI secret-scan step: mandatory, blocks merge on detection.
- Release checklist item: confirm `git ls-files .sparkle-keys/` returns empty before tagging a release.
- The Sparkle private key is referenced only in the release script `scripts/create-release.sh` via an environment variable `SPARKLE_PRIVATE_KEY_PATH` sourced from the local secrets store, never hardcoded.

**Consequences:**
- Release builds can only be signed by engineers with access to the secrets manager entry for the EdDSA private key.
- Automated CI cannot self-sign update archives; a human release step is required to invoke `sign_update` and push the updated `appcast.xml` to the website repo.
- Loss of the private key requires a full key rotation and a forced update that migrates the public key; disaster recovery steps are documented in the release runbook.

---

### ADR-004 — All Release Binaries Notarized with Apple Before Distribution

**Status:** Accepted — RELEASE-BLOCKING (REQ-SEC-03, REQ-REL-PIPE-01)

**Date:** 2026-04-01

**Context:**
macOS Gatekeeper requires that applications distributed outside the Mac App Store be notarized by Apple. Without notarization, Gatekeeper blocks launch with a quarantine warning on macOS 10.15+ and on macOS 15.6 (the minimum supported version) the user cannot override this without disabling Gatekeeper entirely. Distributing an un-notarized app is therefore a hard barrier to adoption and is explicitly prohibited by REQ-SEC-03 and REQ-REL-PIPE-01.

Notarization also requires hardened runtime (ADR-002) and a valid Developer ID Application certificate. The notarization ticket is stapled to the `.app` bundle and the `.dmg` so that offline Gatekeeper checks pass without network access.

**Decision:**
Every release artifact — the `.app` bundle, the `.dmg` installer, and any `.zip` archive published to GitHub Releases — must be notarized using `xcrun notarytool submit` with the Developer ID credentials before distribution. Notarization and ticket stapling are performed as mandatory steps in `scripts/build.sh` and `scripts/create-release.sh`. A build that has not been notarized must never be uploaded to the GitHub release or referenced in `appcast.xml`.

Notarization is performed against Apple's production notarization service (not the legacy `altool`). The Apple ID credentials and app-specific password used for notarization are stored in the macOS Keychain under the profile name `claude-island-notarytool`; `scripts/build.sh` references the profile by name: `--keychain-profile claude-island-notarytool`.

**Compliance control:**
- `scripts/build.sh` calls `xcrun notarytool submit ... --wait` and exits non-zero if the status is not `Accepted`.
- `scripts/build.sh` calls `xcrun stapler staple ClaudeIsland.app` and `xcrun stapler staple ClaudeIsland.dmg` after a successful submission.
- Release checklist item: `xcrun stapler validate ClaudeIsland.dmg` must return `The validate action worked!` before uploading.
- GitHub release artifacts are uploaded only from the output of a completed `scripts/create-release.sh` run; the script enforces the notarization gate.
- The DMG is additionally signed with the Sparkle EdDSA key (ADR-003) before the appcast entry is updated.

**Consequences:**
- An active Apple Developer Program membership and a valid Developer ID Application certificate are required to ship any release.
- Notarization adds 1–5 minutes to the release pipeline; the `--wait` flag is used so the script fails fast on rejection rather than silently proceeding.
- If Apple's notarization service is unavailable, the release is blocked until the service recovers; there is no fallback distribution path.
- The notarization Apple ID credentials must be rotated if the app-specific password is revoked; the Keychain profile `claude-island-notarytool` must be updated accordingly.

---

### ADR-005 — Unix Domain Socket as the Hook-to-App IPC Channel

**Status:** Accepted

**Date:** 2026-04-01

**Context:**
Hook scripts in `~/.claude/hooks/` must deliver session lifecycle events (session start, tool permission requests, session end, `/clear` invocations) to the running Claude Island app process in real time. Candidate IPC mechanisms include: (a) Unix domain socket, (b) named pipe (FIFO), (c) XPC service, (d) distributed notifications, and (e) a local HTTP server.

XPC requires the hook script to be a signed Mach-O binary, which is impractical for shell scripts. Distributed notifications are not suitable for high-frequency structured data. A local HTTP server adds latency and port management complexity. Named pipes lack bidirectional framing needed for approval responses (REQ-PERM-03).

A Unix domain socket supports bidirectional framing, low latency (sub-millisecond on loopback), multiple concurrent connections (one per Claude Code session, REQ-MON-01), and is accessible from shell scripts via `socat` or `nc`. It satisfies the 100 ms UI update latency requirement (REQ-PERF-02) with margin.

**Decision:**
A Unix domain socket at a fixed path (e.g., `~/.claude/claude-island.sock`) is used as the exclusive IPC channel between hook scripts and the app. The app creates and listens on the socket at launch. Hook scripts connect, send a newline-delimited JSON event, and optionally block for a response (for tool approval flows). The app handles socket disconnection and reconnection without requiring a restart (REQ-REL-01, REQ-HOOK-04).

**Consequences:**
- App Sandbox must be disabled (ADR-001) because sandboxed apps cannot create or connect to arbitrary Unix socket paths.
- The socket path must be documented for hook script authors.
- The app must handle concurrent connections from multiple hook processes (one per active Claude Code session).

---

### ADR-006 — Sparkle for Over-the-Air Updates with EdDSA Verification

**Status:** Accepted

**Date:** 2026-04-01

**Context:**
Claude Island is distributed as a direct-download DMG outside the Mac App Store. An auto-update mechanism is required (REQ-UPD-01) to deliver security fixes and features without requiring users to manually download new versions. Sparkle is the de facto standard for macOS DMG-distributed app updates, supports EdDSA (Ed25519) signature verification, and integrates with the hardened runtime (ADR-002) via its XPC-based installer.

**Decision:**
Sparkle (latest release) is the auto-update framework. Updates are fetched from `https://claudeisland.com/appcast.xml` (REQ-UPD-04). Each update archive is signed with the EdDSA private key (ADR-003) and the signature is embedded in `appcast.xml` under `sparkle:edSignature`. The Settings menu displays a green dot indicator when an unseen update is available (REQ-UPD-03). `SUPublicEDKey` in `Info.plist` holds the corresponding public key.

**Consequences:**
- The private key must be protected (ADR-003).
- `appcast.xml` on the website repo must be updated for every release (REQ-REL-PIPE-05) as part of `scripts/create-release.sh`.
- DSA-only signatures are not used; EdDSA is mandatory.

---

### ADR-007 — Developer ID Distribution Certificate (No App Store)

**Status:** Accepted

**Date:** 2026-04-01

**Context:**
App Sandbox is disabled (ADR-001), which disqualifies the app from Mac App Store submission. Distribution must use the Developer ID path: a Developer ID Application certificate signs the app bundle, and the DMG is distributed via direct download and GitHub Releases.

**Decision:**
All release builds are signed with a Developer ID Application certificate. The build setting `CODE_SIGN_IDENTITY = "Developer ID Application"` is set for Release and Distribution configurations in `ClaudeIsland.xcodeproj`. The `scripts/build.sh` passes `--deep` signing for embedded frameworks (Sparkle XPC helpers). No App Store provisioning profiles are used.

**Consequences:**
- Gatekeeper path-randomization (`LSFileQuarantineEnabled`) must not be disabled.
- Notarization (ADR-004) is mandatory for Gatekeeper acceptance.
- A Mac App Store version of the app is not planned; any future App Store target would require a separate entitlements redesign.

---

### ADR-008 — Anonymous Telemetry via Mixpanel, No Conversation Data

**Status:** Accepted

**Date:** 2026-04-01

**Context:**
Understanding aggregate usage patterns (number of active users, frequency of session starts) helps prioritize development. However, Claude Island processes potentially sensitive developer conversation content from JSONL files. Transmitting any conversation content or personally identifiable information would violate user trust and potentially applicable privacy regulations.

**Decision:**
Anonymous telemetry is sent via the Mixpanel SDK (REQ-INT-08). Tracked events are limited to `app_launched` and `session_started` with no properties that identify the user, the project, or the content of any conversation (REQ-SEC-05). No device fingerprint, IP address correlation, or user ID is persisted. The Mixpanel project token is a non-secret value embedded in the binary.

**Consequences:**
- Conversation content, tool names, file paths, and session text must never appear in any Mixpanel event property.
- A privacy policy describing the anonymous telemetry must be published at `https://claudeisland.com/privacy`.
- Users who wish to opt out have no in-app mechanism in the current version; a future ADR must address opt-out if required by applicable law.

---

### ADR-009 — Minimum Deployment Target macOS 15.6, Universal Binary

**Status:** Accepted

**Date:** 2026-04-01

**Context:**
Dynamic Island-style overlay APIs, `SMAppService` for launch-at-login (REQ-SET-03), and relevant window-management APIs are available on macOS 13+. The target user base runs macOS 15.6 (REQ-COMPAT-01) or later. Supporting both Apple Silicon and Intel (REQ-COMPAT-02) maximizes reach during the transition period.

**Decision:**
`MACOSX_DEPLOYMENT_TARGET = 15.6`. The build produces a Universal Binary (`ARCHS = arm64 x86_64`; `ONLY_ACTIVE_ARCH = NO` for Release). All three third-party dependencies (Sparkle, apple/swift-markdown, mixpanel-swift) must provide Universal Binary xcframeworks or be compiled from source as Universal.

**Consequences:**
- APIs deprecated before macOS 15.6 must not be used.
- Non-notch devices are supported via floating-pill fallback mode (REQ-UI-04, REQ-COMPAT-03).
- Intel support may be dropped in a future ADR once Intel Mac market share drops below a threshold to be determined.

---

## 3. Follow-ups

The following items were identified during ADR authoring and require resolution before or shortly after the initial release.

**FU-001 — Automated Test Target (REQ-TEST-01)**
No automated test target currently exists. A unit test target must be added to `ClaudeIsland.xcodeproj`. Priority test coverage: `SessionPhase` state machine transitions (REQ-TEST-02) and hook installation plus socket communication (REQ-TEST-03). Owner: engineering lead. Target: before 1.0 GA.

**FU-002 — Secret Scanning in CI (ADR-003)**
`trufflehog` or an equivalent secret-scanning tool must be integrated into the GitHub Actions workflow as a required status check. This is a prerequisite for the ADR-003 compliance control to be enforced on every pull request. Owner: DevOps. Target: before first public release.

**FU-003 — Telemetry Opt-Out Mechanism**
ADR-008 defers opt-out to a future decision. If distribution reaches users in jurisdictions where anonymous telemetry without an opt-out violates applicable law (e.g., GDPR where anonymous is interpreted strictly), an opt-out preference must be added to the Settings panel and respected before any Mixpanel event is sent. Owner: product + legal review. Target: before EU distribution.

**FU-004 — Notarization Credentials Rotation Runbook**
The notarization Apple ID app-specific password and the Keychain profile `claude-island-notarytool` must have a documented rotation procedure in the release runbook. This runbook must specify the steps to revoke the old password, generate a new one, update the Keychain profile on the release machine, and verify notarization still succeeds before the next release. Owner: release engineer. Target: before 1.0 GA.

**FU-005 — Sparkle Private Key Disaster Recovery**
ADR-003 notes that loss of the EdDSA private key requires a forced-update migration. The disaster-recovery procedure — including how to distribute a key-rotation build to users who cannot receive a normally signed update — must be documented. Owner: engineering lead. Target: before 1.0 GA.

**FU-006 — Intel Deprecation Threshold**
ADR-009 defers the decision on dropping Intel support. A metric (e.g., fewer than 5% of `app_launched` telemetry events from Intel devices for two consecutive quarters) should be defined and reviewed at that time to trigger a new ADR. Owner: product. Target: Q4 2026 review.

**FU-007 — Privacy Policy Publication**
ADR-008 requires a privacy policy at `https://claudeisland.com/privacy` describing the anonymous Mixpanel telemetry. This must be published before the app is distributed publicly. Owner: product. Target: before public launch.

**FU-008 — App Sandbox Re-evaluation**
ADR-001 documents a permanent disablement based on current integration requirements. If Apple introduces entitlements or APIs that allow Unix socket access to arbitrary paths from a sandboxed app, this decision should be revisited. No action required now; flag for re-evaluation at macOS 17.
