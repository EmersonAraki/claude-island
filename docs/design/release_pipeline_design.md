---
codd:
  node_id: design:release-pipeline-design
  type: design
  depends_on:
  - id: governance:decisions
    relation: constrained_by
    semantic: governance
  - id: design:system-design
    relation: depends_on
    semantic: technical
  depended_by:
  - id: detail:component_dependency_map
    relation: depends_on
    semantic: technical
  - id: plan:implementation-plan
    relation: depends_on
    semantic: technical
  conventions:
  - targets:
    - design:release-pipeline-design
    reason: Release binaries must be notarized using Apple notarytool before distribution
      (REQ-SEC-03, REQ-REL-PIPE-01). A DMG must be generated, notarized, and signed
      with the Sparkle EdDSA key (REQ-REL-PIPE-02, REQ-REL-PIPE-03).
  - targets:
    - design:release-pipeline-design
    reason: Sparkle updates must be signed with EdDSA keys that are never committed
      to VCS (REQ-SEC-04, REQ-UPD-02). The appcast at https://claudeisland.com/appcast.xml
      must be updated per release (REQ-UPD-04, REQ-REL-PIPE-05).
  - targets:
    - design:release-pipeline-design
    reason: All release builds must use the Developer ID distribution certificate
      and hardened runtime (REQ-BUILD-02). Release without this signing configuration
      is rejected by Apple notarization.
---

# Release Pipeline Design

## 1. Overview

The Claude Island release pipeline produces a signed, notarized, and Sparkle-signed DMG artifact that can be safely distributed to end users via GitHub Releases and direct download. The pipeline is implemented across two shell scripts — `scripts/build.sh` and `scripts/create-release.sh` — and enforces a strict sequence of gates that must all pass before any artifact is uploaded or referenced in `appcast.xml`.

All release artifacts target macOS 15.6 as the minimum deployment target (`MACOSX_DEPLOYMENT_TARGET = 15.6`) and are built as Universal Binaries (`ARCHS = arm64 x86_64`, `ONLY_ACTIVE_ARCH = NO`). Distribution via the Mac App Store is not supported because App Sandbox is permanently disabled (ADR-001); all distribution is via Developer ID.

**Release-blocking constraints this document enforces:**

| ID | Constraint | Source |
|---|---|---|
| REQ-REL-PIPE-01 | Every release binary must be notarized using `xcrun notarytool` before distribution | ADR-004, REQ-SEC-03 |
| REQ-REL-PIPE-02 | A DMG must be generated and notarized for each release | ADR-004 |
| REQ-REL-PIPE-03 | The DMG must be signed with the Sparkle EdDSA key before the appcast entry is written | ADR-003, ADR-006 |
| REQ-REL-PIPE-05 | `https://claudeisland.com/appcast.xml` must be updated as the final step of every release | ADR-006, REQ-UPD-04 |
| REQ-BUILD-02 | All release builds must use the Developer ID Application certificate with hardened runtime enabled | ADR-002, ADR-007 |
| REQ-SEC-03 | Shipping without notarization is release-blocking; Gatekeeper will block launch on macOS 15.6+ | ADR-004 |
| REQ-SEC-04 | Sparkle EdDSA private keys must never be committed to version control | ADR-003 |
| REQ-UPD-02 | Sparkle updates must carry a valid EdDSA signature; DSA-only signatures are rejected | ADR-006 |

A build that fails any gate in `scripts/build.sh` must exit non-zero and must not produce uploadable artifacts. `scripts/create-release.sh` must not upload to GitHub Releases or push to the website `appcast.xml` repository unless `scripts/build.sh` completes without error.

---

## 2. Architecture

### 2.1 Pipeline Stages

The release pipeline executes in eight sequential stages. No stage may be skipped; each stage's output is the input to the next.

```
Stage 1: Clean Build
        │  xcodebuild archive
        │  ARCHS=arm64 x86_64 / ONLY_ACTIVE_ARCH=NO
        │  MACOSX_DEPLOYMENT_TARGET=15.6
        ▼
Stage 2: Code Sign (Developer ID)
        │  codesign --deep --force
        │  CODE_SIGN_IDENTITY="Developer ID Application"
        │  ENABLE_HARDENED_RUNTIME=YES (bit 0x10000)
        │  Entitlements: app-sandbox=false
        ▼
Stage 3: DMG Creation
        │  hdiutil create ClaudeIsland.dmg
        │  from signed .app bundle
        ▼
Stage 4: Notarization
        │  xcrun notarytool submit --keychain-profile
        │      claude-island-notarytool --wait
        │  Exit non-zero if status != "Accepted"
        ▼
Stage 5: Staple Tickets
        │  xcrun stapler staple ClaudeIsland.app
        │  xcrun stapler staple ClaudeIsland.dmg
        │  xcrun stapler validate ClaudeIsland.dmg
        │      → must return "The validate action worked!"
        ▼
Stage 6: Sparkle EdDSA Signing
        │  sign_update ClaudeIsland.dmg
        │      using $SPARKLE_PRIVATE_KEY_PATH
        │  Captures sparkle:edSignature value
        ▼
Stage 7: GitHub Release Upload
        │  gh release create vX.Y.Z
        │  Upload ClaudeIsland.dmg + ClaudeIsland.app.zip
        │  All artifacts must be notarized (stage 5 gate)
        ▼
Stage 8: Appcast Update
        │  Inject new <item> into appcast.xml
        │      sparkle:edSignature from stage 6
        │      SUFeedURL = https://claudeisland.com/appcast.xml
        │  Push to website repository
```

### 2.2 Build Configuration

`scripts/build.sh` invokes `xcodebuild` with the following configuration for all release artifacts:

```
xcodebuild archive \
  -project ClaudeIsland.xcodeproj \
  -scheme ClaudeIsland \
  -configuration Release \
  -archivePath build/ClaudeIsland.xcarchive \
  ARCHS="arm64 x86_64" \
  ONLY_ACTIVE_ARCH=NO \
  MACOSX_DEPLOYMENT_TARGET=15.6 \
  ENABLE_HARDENED_RUNTIME=YES \
  CODE_SIGN_IDENTITY="Developer ID Application"
```

After archiving, `xcodebuild -exportArchive` is invoked with an export options plist specifying `method = developer-id` and `signingStyle = manual`. `--deep` code signing is applied to cover the embedded Sparkle XPC helpers.

### 2.3 Entitlements Compliance (REQ-BUILD-02, ADR-001, ADR-002)

The entitlements file `ClaudeIsland.entitlements` must satisfy two release-blocking conditions verified by CI before code signing:

1. **App Sandbox disabled (REQ-SEC-01, ADR-001):** The key `com.apple.security.app-sandbox` must be absent or set to `false`. CI step: `codesign -d --entitlements - ClaudeIsland.app | grep -c "app-sandbox.*true"` must return `0`.

2. **Hardened runtime enabled (REQ-BUILD-02, ADR-002):** CI step: `codesign -dv ClaudeIsland.app 2>&1 | grep flags | grep -E "0x[0-9a-f]*1[0-9a-f]{4}"` must match (bit `0x10000` set). The `scripts/build.sh` `codesign` invocation explicitly passes `--options runtime`.

No JIT entitlement (`cs.allow-jit`), no library-validation disable (`cs.disable-library-validation`), and no `cs.allow-dyld-environment-variables` may be present without a new ADR. Presence of any of these entitlements triggers a CI failure before notarization is attempted.

### 2.4 Notarization Gate (REQ-SEC-03, REQ-REL-PIPE-01, ADR-004)

Notarization is performed against Apple's production notarization service using `notarytool` (not the deprecated `altool`). The Apple ID credentials are stored in the macOS Keychain under the profile name `claude-island-notarytool` on the release engineer's machine; they are never hardcoded in scripts.

```bash
xcrun notarytool submit build/ClaudeIsland.dmg \
  --keychain-profile claude-island-notarytool \
  --wait

# Exit immediately if status is not "Accepted"
if [ $? -ne 0 ]; then
  echo "Notarization failed — aborting release" >&2
  exit 1
fi
```

The `--wait` flag ensures the script blocks until Apple returns a final status; a non-`Accepted` result exits non-zero, halting `create-release.sh` before any upload. There is no fallback distribution path if Apple's notarization service is unavailable; the release is blocked until the service recovers.

After a successful `Accepted` response, ticket stapling is performed:

```bash
xcrun stapler staple build/ClaudeIsland.app
xcrun stapler staple build/ClaudeIsland.dmg
xcrun stapler validate build/ClaudeIsland.dmg
```

`stapler validate` must return `The validate action worked!`. Any other output causes `scripts/build.sh` to exit non-zero. This ensures offline Gatekeeper checks pass without network access (ADR-004).

### 2.5 Sparkle EdDSA Signing (REQ-REL-PIPE-03, REQ-SEC-04, REQ-UPD-02, ADR-003, ADR-006)

The Sparkle EdDSA private key is stored exclusively in `.sparkle-keys/` on the release engineer's local machine and in the secrets manager. The `.sparkle-keys/` directory is listed in `.gitignore` and `.git/info/exclude`. The key is never committed to version control (REQ-SEC-04, ADR-003). `scripts/create-release.sh` references it via the environment variable `SPARKLE_PRIVATE_KEY_PATH`, which the release engineer sets by sourcing from their local secrets store before invoking the script.

After the notarization gate passes, `sign_update` (Sparkle CLI tool) signs the DMG:

```bash
SIGNATURE=$(sign_update build/ClaudeIsland.dmg "$SPARKLE_PRIVATE_KEY_PATH")
```

The resulting `sparkle:edSignature` value is captured and injected into the new `<item>` block written to `appcast.xml`. DSA-only signatures are not used; the EdDSA signature is mandatory (REQ-UPD-02, ADR-006).

The corresponding Ed25519 public key is embedded in `Info.plist` under `SUPublicEDKey` and is safe to commit. Sparkle verifies the signature on the client side against this public key before installing any update; an update with an invalid or missing signature is rejected and the user is notified.

Key rotation uses `generate_keys` (Sparkle CLI tool) to produce a new Ed25519 keypair, updates `SUPublicEDKey` in `Info.plist`, re-signs all appcast entries with `sign_update`, revokes the old key from the secrets manager, and records the rotation date and reason in a new ADR addendum (ADR-003 key rotation procedure).

### 2.6 Secret Scanning (ADR-003, REQ-SEC-04)

CI enforces two complementary controls to prevent the Sparkle private key from entering version control:

1. **Pre-commit hook:** Asserts that no file matching `*.key`, `*.pem`, `*.p8`, `private_key`, or `sparkle_private*` is staged. The hook is installed by `scripts/setup-dev.sh` and blocks the commit if any matching file is found.

2. **CI secret scan:** A `trufflehog` scan (or equivalent with equivalent detection coverage) runs as a required status check on every pull request targeting `main`. Merge is blocked if a private key pattern is detected in any file or commit in the PR branch. This satisfies ADR-003 compliance control and FU-002.

Before every release tag is created, `scripts/create-release.sh` asserts `git ls-files .sparkle-keys/` returns empty. Any output from that command halts the release.

### 2.7 Appcast Update (REQ-UPD-04, REQ-REL-PIPE-05, ADR-006)

The final stage of `scripts/create-release.sh` updates `appcast.xml` at `https://claudeisland.com/appcast.xml`. The script generates a new `<item>` element containing:

- `<title>` — version string (e.g., `Version 1.3`)
- `<sparkle:version>` — CFBundleVersion (e.g., `13`)
- `<sparkle:shortVersionString>` — CFBundleShortVersionString (e.g., `1.3`)
- `<sparkle:edSignature>` — Ed25519 signature produced by `sign_update` in stage 6
- `<length>` — byte size of the DMG
- `<url>` — direct download URL for `ClaudeIsland.dmg` on GitHub Releases
- `<sparkle:minimumSystemVersion>` — `15.6`
- `<pubDate>` — RFC 2822 formatted release date

The appcast is committed to the website repository and pushed. No GitHub Release artifact may be referenced in `appcast.xml` before it has passed the notarization gate (stage 4–5) and carries a valid EdDSA signature (stage 6). Updating `appcast.xml` before these gates pass is a release-blocking error.

The Sparkle `SUFeedURL` in the app's `Info.plist` is `https://claudeisland.com/appcast.xml`. Sparkle polls this URL on a configurable schedule and presents updates through the auto-update subsystem (section 2.13 of the system design). The Settings panel displays a green dot indicator when an unseen update is available (AC-UPD-03).

### 2.8 GitHub Release Upload

After the stapler validation passes, `scripts/create-release.sh` uploads artifacts to the GitHub Release using `gh release create`:

```bash
gh release create "v${VERSION}" \
  --title "Claude Island ${VERSION}" \
  --notes-file release-notes.md \
  build/ClaudeIsland.dmg \
  build/ClaudeIsland.app.zip
```

Both the `.dmg` and the `.zip` archive must have been notarized and stapled (stages 4–5) before upload. The script asserts the `xcrun stapler validate` gate passed for the `.dmg` before invoking `gh`. The `.zip` archive is an additional distribution format for users who prefer zip-based Sparkle delta updates; it is signed with the same EdDSA key.

### 2.9 CI Integration

The GitHub Actions workflow runs the following required status checks on every pull request targeting `main`:

| Check | Failure action |
|---|---|
| `trufflehog` secret scan | Blocks merge |
| Entitlements assertion (`app-sandbox = false`) | Blocks merge |
| Hardened runtime flag assertion | Blocks merge |
| Universal Binary verification (`lipo -info`) | Blocks merge |
| Unit test suite (SessionPhase state machine, AC-TEST-02) | Blocks merge |
| Hook/socket integration tests (AC-TEST-03) | Blocks merge |

CI does not perform notarization on pull requests because it requires the `claude-island-notarytool` Keychain profile present only on the release engineer's machine. Notarization is a local release step executed by the release engineer after merging.

### 2.10 Release Engineer Responsibilities

The release engineer must complete the following checklist before tagging a release. Each item is a release-blocking gate; failure of any item must halt the release:

1. Confirm `git ls-files .sparkle-keys/` returns empty.
2. Confirm `codesign -d --entitlements - ClaudeIsland.app` does not include `app-sandbox = true`.
3. Confirm `codesign -dv ClaudeIsland.app` shows the hardened-runtime flag (bit `0x10000`).
4. Confirm `spctl -a -vvv ClaudeIsland.app` returns `source=Developer ID`.
5. Confirm `xcrun stapler validate ClaudeIsland.dmg` returns `The validate action worked!`.
6. Confirm `SPARKLE_PRIVATE_KEY_PATH` is set from the local secrets store before invoking `create-release.sh`.
7. Confirm `appcast.xml` `<sparkle:edSignature>` matches the output of `sign_update` for the uploaded DMG.
8. Confirm the GitHub Release artifact URLs in `appcast.xml` resolve to the correct version.

### 2.11 Privacy and Legal Compliance

The anonymous Mixpanel telemetry embedded in release builds (ADR-008) requires a privacy policy at `https://claudeisland.com/privacy` to be published before any public distribution (FU-007). The release pipeline does not gate on this URL being live, but the release runbook records it as a prerequisite. No conversation content, file paths, tool names, user IDs, or device fingerprints may appear in any Mixpanel event property; this is verified by auditing all `AnalyticsClient` call sites before release (AC-SEC-03, FC-SEC-02).

If the app is distributed to users in EU jurisdictions, a telemetry opt-out preference must be added to the Settings panel and respected before any `AnalyticsClient` event is sent. The Mixpanel call-site boundary (isolated behind `MixpanelBridge`) supports this by gating invocations on a `UserDefaults` flag, but the Settings item does not yet exist (OQ-001 / FU-003).

### 2.12 Notarization Credential Management

The Apple ID app-specific password for notarization is stored in the macOS Keychain under the profile name `claude-island-notarytool`. This profile must be configured on the release engineer's machine with:

```bash
xcrun notarytool store-credentials claude-island-notarytool \
  --apple-id "<release-engineer-apple-id>" \
  --team-id "<TEAM_ID>" \
  --password "<app-specific-password>"
```

If the app-specific password is revoked, the profile must be updated and notarization verified on a test build before the next production release (FU-004). The rotation runbook must document: revoking the old password at appleid.apple.com, generating a new app-specific password, running `notarytool store-credentials` to update the Keychain profile, and performing a dry-run notarization against a test binary to confirm the new credentials work.

---

## 3. Open Questions

**OQ-PIPE-001 — Notarization Credentials Rotation Runbook (FU-004)**
A documented step-by-step procedure for revoking and regenerating the app-specific password for the `claude-island-notarytool` Keychain profile does not yet exist in the release runbook. The procedure must cover: revocation at appleid.apple.com, new app-specific password generation, `notarytool store-credentials` update, dry-run notarization verification, and confirmation that `xcrun stapler validate` still passes. This must be completed before 1.0 GA.

**OQ-PIPE-002 — Sparkle Private Key Disaster Recovery (FU-005)**
If the EdDSA private key is lost or compromised, users running old versions cannot receive a normally signed update because the public key embedded in their installed binary no longer matches a new key pair. A forced-migration strategy — either a transitional build signed with both the old and new keys, or an out-of-band side-channel distribution — must be designed and documented in the release runbook before 1.0 GA.

**OQ-PIPE-003 — CI Notarization for Nightly or Staging Builds**
The current pipeline performs notarization exclusively as a local release engineer step because the `claude-island-notarytool` Keychain profile is not available in CI. If a nightly or staging distribution channel is introduced in the future, a CI-accessible credential store (e.g., GitHub Actions secrets mapped to an App Store Connect API key used with `notarytool --key`) would be required. A new ADR must evaluate the security implications of storing notarization credentials in CI secrets versus maintaining the manual-only gate.

**OQ-PIPE-004 — Delta Update Generation**
The current pipeline produces only full-update DMG and zip artifacts. Sparkle supports binary delta updates (`sparkle:deltaFrom` entries in `appcast.xml`) that significantly reduce download size for incremental releases. No delta generation step exists in `scripts/create-release.sh`. If delta updates are added, `BinaryDelta` (Sparkle CLI tool) must be invoked for each supported delta path, each delta archive must be independently signed with `sign_update`, and `appcast.xml` must carry separate `<enclosure>` entries per delta. This must be formally designed before implementation to avoid appcast schema errors.

**OQ-PIPE-005 — Appcast Hosting and CDN Availability**
`appcast.xml` is served from `https://claudeisland.com/appcast.xml`. If the website hosting is unavailable, Sparkle cannot check for updates and users receive no update notifications (no timeout fallback is specified). The website's uptime SLA and the CDN or hosting provider used have not been formally documented. A downtime event does not affect installed app function but does prevent update delivery; the risk tolerance for this should be recorded.

**OQ-PIPE-006 — Release Artifact Retention Policy**
GitHub Releases retain uploaded artifacts indefinitely by default. No retention or deletion policy for old release DMGs referenced in historical `appcast.xml` entries has been defined. If old artifact URLs break, users running very old versions of the app who trigger a Sparkle update check against an appcast entry pointing to a deleted file will receive a download error. A retention policy and the handling of obsolete appcast entries must be documented.
