---
name: xcode-ci-doctor
description: >-
  Operational troubleshooting manual for debugging Apple platforms and Xcode CI/CD pipelines. Use when: resolving xcodebuild failures or exit code 65 on GitHub Actions (macos-14/macos-15), fixing 'project format version 110 is not supported' errors by downgrading to objectVersion 70, bypassing headless code signing requirements (CODE_SIGNING_ALLOWED=NO), locating unshared schemes in xcshareddata, setting up simulator destination specifiers, or implementing XCTSkipIf for hardware-dependent tests (Metal/CoreML/Vision) on cloud virtualization VMs.
---

# Xcode CI Doctor (Headless Apple Pipeline Troubleshooting) 🍏🛠️

The definitive operational manual for AI coding agents tasked with configuring, diagnosing, fixing, and maintaining headless macOS and iOS Continuous Integration (CI) runners.

---

## 1. Executive Summary & Core Philosophy

Headless Apple CI environments (such as GitHub Actions `macos-14`, `macos-15`, GitLab macOS executors, and Xcode Cloud) differ drastically from local developer workstations running interactive Xcode GUIs:

1. **Common Failure Modes**:
   - **Interactive Prompt Hangs**: Build commands wait forever for keychain unlock passwords or interactive code signing dialogs.
   - **Toolchain / Project Format Mismatches**: Xcode versions running on local machines (e.g. Xcode 27 / macOS Sequoia beta) generate project files (`objectVersion = 110`) that cannot be opened by standard CI runner versions (e.g. Xcode 16 / `objectVersion = 70`).
   - **Missing Shared Schemes**: Schemes created locally default to `xcuserdata/` (gitignored), so `xcodebuild` fails on CI with `The project does not contain a scheme named 'X'`.
   - **Missing Hardware Acceleration**: Tests utilizing Apple Neural Engine (CoreML), GPU-accelerated Vision, or physical audio devices fail on headless virtualization hypervisors.
   - **Simulator Destination Failures**: Generic destination specifiers fail because exact simulator versions differ across runner images.

2. **The Doctor's Mandate**:
   - **Zero Interactive Blocks**: All signing, provisioning, and keychain interactions must be explicitly bypassed or automated via headless arguments.
   - **Backwards-Compatible Project Formats**: Never push Xcode project versions higher than the maximum supported version of your production CI runner image.
   - **Graceful Hardware Fallbacks**: Unit test suites must detect virtualization environments and use `XCTSkipIf` or software mock fallbacks instead of crashing.
   - **Deterministic Simulator Targeting**: Use platform and OS wildcards (`platform=iOS Simulator,name=iPhone 15`) rather than brittle UDIDs.

---

## 2. Headless Pipeline Mental Model & Architecture

```
+-----------------------------------------------------------------------------------+
|                        APPLE HEADLESS CI ARCHITECTURE                             |
+-----------------------------------------------------------------------------------+
| 1. RUNNER IMAGE SPECIFICATION                                                     |
|    - macos-14 (Sonoma / Xcode 15 & 16) | macos-15 (Sequoia / Xcode 16)            |
+-----------------------------------------------------------------------------------+
| 2. XCODE VERSION SELECTOR (DEVELOPER_DIR)                                         |
|    - sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer         |
+-----------------------------------------------------------------------------------+
| 3. CODE SIGNING BYPASS FLAGS                                                      |
|    - CODE_SIGNING_ALLOWED=NO CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO       |
+-----------------------------------------------------------------------------------+
| 4. SCHEME VISIBILITY                                                              |
|    - xcshareddata/xcschemes/<Scheme>.xcscheme (MUST be tracked in Git)           |
+-----------------------------------------------------------------------------------+
| 5. DESTINATION SPECIFIERS                                                         |
|    - -destination 'platform=macOS' OR 'platform=iOS Simulator,name=iPhone 15'     |
+-----------------------------------------------------------------------------------+
```

---

## 3. Systematic Diagnostic Protocol for Build Failures

When an Apple CI workflow turns red, follow this 5-stage triage protocol:

```
[Stage 1: Log Parsing]
       │
       ▼
  Identify failure category: Scheme, Code Signing, Project Format, Destination, or Hardware Test
       │
       ▼
[Stage 2: Environment Audit]
       │
       ▼
  Check runner OS (`sw_vers`), selected Xcode (`xcodebuild -version`), and available SDKs
       │
       ▼
[Stage 3: Repository State Verification]
       │
       ▼
  Verify .xcscheme is in git, objectVersion in project.pbxproj <= 70, entitlements sanitized
       │
       ▼
[Stage 4: Surgical Patch Application]
       │
       ▼
  Apply targeted flag, script, or configuration change
       │
       ▼
[Stage 5: Local Headless Reproduction]
       │
       ▼
  Run command locally with empty keychain and headless flags before pushing
```

---

## 4. Comprehensive Catalog of 25+ Xcode CI Errors & Exact Fixes

### Error 01: Project Format Version Incompatibility (objectVersion 110 vs 70)
- **Log Output**:
  `error: The project format version 110 is not supported by this version of Xcode.`
- **Root Cause**:
  The project was saved in Xcode 27+ on a local workstation, writing `objectVersion = 110`. The CI runner uses Xcode 16 (which only supports `objectVersion <= 77` / format 70).
- **The Doctor's Fix**:
  Edit `project.pbxproj` and downgrade the format headers:
  ```sed
  sed -i '' 's/objectVersion = 110;/objectVersion = 70;/g' MyApp.xcodeproj/project.pbxproj
  sed -i '' 's/LastSwiftUpdateCheck = 2700;/LastSwiftUpdateCheck = 1600;/g' MyApp.xcodeproj/project.pbxproj
  sed -i '' 's/CreatedOnToolsVersion = 27.0;/CreatedOnToolsVersion = 16.0;/g' MyApp.xcodeproj/project.pbxproj
  ```

---

### Error 02: Scheme Not Found on CI Runner
- **Log Output**:
  `xcodebuild: error: The project named "MyApp" does not contain a scheme named "MyApp".`
- **Root Cause**:
  The scheme was created in the user's private data folder (`MyApp.xcodeproj/xcuserdata/.../xcschemes/MyApp.xcscheme`) which is ignored by `.gitignore`.
- **The Doctor's Fix**:
  Move or recreate the scheme inside the shared data directory:
  ```bash
  mkdir -p MyApp.xcodeproj/xcshareddata/xcschemes/
  cp MyApp.xcodeproj/xcuserdata/*.xcuserdatad/xcschemes/MyApp.xcscheme      MyApp.xcodeproj/xcshareddata/xcschemes/
  git add MyApp.xcodeproj/xcshareddata/xcschemes/MyApp.xcscheme
  ```

---

### Error 03: Signing Requires a Development Team
- **Log Output**:
  `error: Signing for "MyApp" requires a development team. Select a development team in the Signing & Capabilities editor.`
- **The Doctor's Fix**:
  Inject headless signing bypass flags in the CI workflow:
  ```bash
  xcodebuild build     -project MyApp.xcodeproj     -scheme MyApp     -destination 'platform=macOS'     CODE_SIGNING_ALLOWED=NO     CODE_SIGN_IDENTITY=""     CODE_SIGNING_REQUIRED=NO     CODE_SIGN_ENTITLEMENTS=""
  ```

---

### Error 04: Destination Is Not Valid for Scheme
- **Log Output**:
  `xcodebuild: error: Unable to find a destination matching the provided destination specifier: { platform:iOS Simulator, OS:latest, name:iPhone 14 }`
- **Root Cause**:
  Exact device name or OS version does not exist on the runner's preinstalled simulator runtimes.
- **The Doctor's Fix**:
  Query available runtimes dynamically or use generic architecture specifiers:
  ```bash
  # Query available simulator:
  xcrun simctl list devices available
  
  # Safe generic destination:
  -destination 'platform=iOS Simulator,name=iPhone 15'
  ```

---

### Error 05: CoreML / Vision Test Failure on Headless VM
- **Log Output**:
  `Test Case '-[VisionTests testDocumentDetection]' failed (0.05s). Error: Failed to allocate GPU texture; Metal device unavailable.`
- **Root Cause**:
  Headless cloud VMs lack a physical GPU and Neural Engine. Metal pipelines throw hardware allocation errors.
- **The Doctor's Fix**:
  Use `XCTSkipIf` with virtualization detection:
  ```swift
  import XCTest

  final class VisionPipelineTests: XCTestCase {
      func testMetalAcceleration() throws {
          #if targetEnvironment(simulator)
          // Simulator software fallback
          #else
          let isHeadlessCI = ProcessInfo.processInfo.environment["CI"] != nil
          try XCTSkipIf(isHeadlessCI, "Skipping hardware Metal test on headless CI runner")
          #endif
          
          let result = try runMetalPipeline()
          XCTAssertNotNil(result)
      }
  }
  ```

---

### Error 06: Package Resolution Times Out (SPM Hang)
- **Log Output**:
  `Fetching dependencies... Operation timed out after 300 seconds.`
- **The Doctor's Fix**:
  Cache `~/.swiftpm/cache` and `.build` directory in GitHub Actions:
  ```yaml
  - name: Cache SwiftPM Dependencies
    uses: actions/cache@v4
    with:
      path: .build
      key: spm-${{ runner.os }}-${{ hashFiles('Package.resolved') }}
      restore-keys: |
        spm-${{ runner.os }}-
  ```

---

### Error 07: Keychain Prompt Blocks Headless Runner
- **Log Output**:
  `User interaction is not allowed. (Command /usr/bin/codesign failed with exit code 1)`
- **The Doctor's Fix**:
  If signing is genuinely required for distribution artifacts, create and unlock a temporary headless keychain:
  ```bash
  security create-keychain -p "$KEYCHAIN_PWD" build.keychain
  security default-keychain -s build.keychain
  security unlock-keychain -p "$KEYCHAIN_PWD" build.keychain
  security set-keychain-settings -t 3600 -u build.keychain
  security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PWD" build.keychain
  ```

---

### Error 08: Missing Swift Concurrency Import Visibility Flag
- **Log Output**:
  `error: member 'somePublisher' cannot be used without explicit 'import Combine'`
- **Root Cause**:
  Xcode 16+ enforces `SWIFT_UPCOMING_FEATURE_MEMBER_IMPORT_VISIBILITY = YES`.
- **The Doctor's Fix**:
  Ensure every file using `@Published`, `@AppStorage`, or Combine explicitly writes `import Combine` at the top.

---

### Error 09: Swift Testing Library Missing on Older macOS Runners
- **Log Output**:
  `error: no such module 'Testing'`
- **The Doctor's Fix**:
  Ensure CI runner is `macos-14` with Xcode 16 selected via `DEVELOPER_DIR`, or gate test files:
  ```swift
  #if canImport(Testing)
  import Testing
  // Swift Testing suite
  #endif
  ```

---

### Error 10: DerivedData Path Permissions Error
- **The Doctor's Fix**:
  Direct output to an isolated local folder:
  ```bash
  xcodebuild -derivedDataPath .build/DerivedData ...
  ```

---

### Error 11: Podfile / CocoaPods Lock Mismatch in CI
- **The Doctor's Fix**:
  Run `bundle exec pod install --repo-update` or migrate target to modern Swift Package Manager.

---

### Error 12: Architecture Mismatch on Apple Silicon Runners
- **Log Output**:
  `ld: symbol(s) not found for architecture x86_64`
- **The Doctor's Fix**:
  Enforce `ARCHS=arm64` and run on native Apple Silicon runners (`macos-14` / `macos-15`).

---

### Error 13: xcpretty Ruby Dependency Crash
- **The Doctor's Fix**:
  Use native formatting flags or `xcodebuild -quiet` rather than unmaintained Ruby gems.

---

### Error 14: Simulator Audio Daemon Crash
- **The Doctor's Fix**:
  Mock `AVAudioSession` interactions during unit test runs.

---

### Error 15: Swift Package Member Visibility Build Settings
- **The Doctor's Fix**:
  Pass explicit build configuration in `Package.swift`:
  ```swift
  swiftSettings: [
      .enableUpcomingFeature("MemberImportVisibility")
  ]
  ```

---

## 5. Master GitHub Actions Workflow for Xcode Apps

Here is the bulletproof, production-grade GitHub Actions template for iOS and macOS apps:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-test:
    name: Build & Test (${{ matrix.platform }})
    runs-on: macos-14
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macOS
            destination: 'platform=macOS'
            scheme: MyApp-macOS
          - platform: iOS
            destination: 'platform=iOS Simulator,name=iPhone 15'
            scheme: MyApp-iOS

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Select Xcode 16
        run: |
          sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer
          xcodebuild -version

      - name: Cache DerivedData
        uses: actions/cache@v4
        with:
          path: .derivedData
          key: derived-data-${{ matrix.platform }}-${{ hashFiles('**/*.swift', '**/*.pbxproj') }}
          restore-keys: |
            derived-data-${{ matrix.platform }}-

      - name: Build Application
        run: |
          xcodebuild build \
            -project MyApp.xcodeproj \
            -scheme ${{ matrix.scheme }} \
            -destination '${{ matrix.destination }}' \
            -derivedDataPath .derivedData \
            CODE_SIGNING_ALLOWED=NO \
            CODE_SIGN_IDENTITY="" \
            CODE_SIGNING_REQUIRED=NO \
            COMPILER_INDEX_STORE_ENABLE=NO

      - name: Run Test Suite
        env:
          CI: true
        run: |
          xcodebuild test \
            -project MyApp.xcodeproj \
            -scheme ${{ matrix.scheme }} \
            -destination '${{ matrix.destination }}' \
            -derivedDataPath .derivedData \
            CODE_SIGNING_ALLOWED=NO \
            CODE_SIGN_IDENTITY="" \
            CODE_SIGNING_REQUIRED=NO
```

---

## 6. Anti-Patterns Compendium (15 Fatal CI Mistakes)

| Anti-Pattern | Why It Fails | Correct Solution |
| :--- | :--- | :--- |
| **1. Forgetting to share schemes** | CI fails with `Scheme not found`. | Commit `MyApp.xcscheme` to `xcshareddata/xcschemes/`. |
| **2. Pushing objectVersion 110** | Xcode 16 on GitHub Actions cannot read Xcode 27 files. | Downgrade `objectVersion = 70`. |
| **3. Hardcoding simulator UDIDs** | UDIDs change on every VM spin-up. | Use `name=iPhone 15` or dynamic lookup. |
| **4. Requiring real certificates in PR builds** | Forked PRs cannot access organization secrets; build fails. | Use `CODE_SIGNING_ALLOWED=NO`. |
| **5. Testing hardware-accelerated ML on VMs** | Metal textures fail to allocate; tests crash. | Use `XCTSkipIf(ProcessInfo.processInfo.environment["CI"] != nil)`. |
| **6. Storing DerivedData in system default path** | Cache actions cannot preserve `/Library/Developer/DerivedData`. | Specify `-derivedDataPath .derivedData`. |
| **7. Not using `concurrency: cancel-in-progress`** | Rapid commits exhaust Mac runner minutes. | Add `concurrency` cancellation block to YAML. |
| **8. Using deprecated `macos-12` or `macos-13`** | Runners sunsetted; builds fail to initialize. | Use `macos-14` or `macos-15`. |
| **9. Invoking GUI dialogs in pre-build scripts** | Headless runner freezes indefinitely waiting for clicks. | Ensure all custom scripts run headless (`-batch` mode). |
| **10. Ignoring missing `Package.resolved`** | Builds resolve differing dependency versions across runs. | Always commit `Package.resolved`. |
| **11. Cluttering logs with full xcodebuild traces** | Traces exceed GitHub Actions 4MB log limit. | Use `-quiet` or filter compiler output. |
| **12. Relying on local CocoaPods cache** | CocoaPods fails on CI with out-of-date spec repos. | Prefer SPM or run `--repo-update`. |
| **13. Running tests without parallel flags** | Sequential test runs waste expensive Mac runner minutes. | Enable parallel testing in test plans. |
| **14. Forgetting `COMPILER_INDEX_STORE_ENABLE=NO`** | CI indexes code that will be discarded on VM teardown. | Disable index store to save 25% build time. |
| **15. Not sanitizing environment secrets in error dumps** | CI failure logs print environment variables to the web. | Never dump raw shell environments in failure traps. |

---

## 7. Deep-Dive: Diagnostic Case Studies

### Case Study 01: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #1 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module1.swift
error: Signing for "MyApp_Submodule1" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module1 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 02: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #2 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module2.swift
error: Signing for "MyApp_Submodule2" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module2 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 03: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #3 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module3.swift
error: Signing for "MyApp_Submodule3" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module3 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 04: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #4 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module4.swift
error: Signing for "MyApp_Submodule4" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module4 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 05: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #5 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module5.swift
error: Signing for "MyApp_Submodule5" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module5 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 06: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #6 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module6.swift
error: Signing for "MyApp_Submodule6" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module6 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 07: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #7 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module7.swift
error: Signing for "MyApp_Submodule7" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module7 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 08: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #8 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module8.swift
error: Signing for "MyApp_Submodule8" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module8 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 09: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #9 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module9.swift
error: Signing for "MyApp_Submodule9" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module9 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 10: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #10 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module10.swift
error: Signing for "MyApp_Submodule10" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module10 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 11: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #11 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module11.swift
error: Signing for "MyApp_Submodule11" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module11 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 12: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #12 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module12.swift
error: Signing for "MyApp_Submodule12" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module12 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 13: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #13 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module13.swift
error: Signing for "MyApp_Submodule13" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module13 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 14: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #14 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module14.swift
error: Signing for "MyApp_Submodule14" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module14 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.


### Case Study 15: Headless Pipeline Triage Scenario

#### Scenario Overview
A developer pushed a feature update for Apple sub-module #15 (e.g. background sync, image processing, or Keychain storage). The local build succeeded in Xcode GUI, but the GitHub Actions `macos-14` workflow terminated with an exit code 65.

#### The Error Log
```
CompileSwift normal arm64 /Users/runner/work/app/Sources/Module15.swift
error: Signing for "MyApp_Submodule15" requires a development team.
Select a development team in the Signing & Capabilities editor.
** BUILD FAILED ** [Exit Code 65]
```

#### Diagnostic Breakdown
1. **Local vs CI Isolation**: The local engineer had a personal Apple Developer Certificate installed in their local login keychain. Xcode automatically applied automatic signing.
2. **Headless Reality**: The CI runner is a clean virtual machine with no signing identities.
3. **Target Analysis**: The target was being compiled with default `CODE_SIGNING_REQUIRED=YES`.

#### Automated Doctor Resolution
Apply the headless signing override:
```bash
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme Module15 \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

#### Verification Check
- Build succeeds in 12 seconds with zero code signing prompts.
- Test suite executes cleanly without keychain unlock locks.

## 8. Headless xcodebuild CLI Flags Reference

| Flag / Parameter | Purpose in CI | Recommended CI Setting |
| :--- | :--- | :--- |
| `CODE_SIGNING_ALLOWED=NO` | Bypasses local and CI certificate signing requirements | Always set to `NO` on pull request and test workflows |
| `CODE_SIGN_IDENTITY=""` | Clears signing identity override | Pass empty string `""` to prevent Xcode from searching keychain |
| `CODE_SIGNING_REQUIRED=NO` | Disables codesign requirement checks | Always set to `NO` for headless unit testing |
| `-derivedDataPath <path>` | Isolates build artifacts into repository local folder | Set to `.derivedData` to facilitate caching with actions/cache |
| `-skipPackagePluginValidation` | Prevents Xcode 15/16 SPM command plugin trust dialog prompts | Always pass when using Swift packages with build tool plugins |
| `-disableAutomaticPackageResolution` | Prevents xcodebuild from mutating `Package.resolved` during build | Pass when strict reproducible dependency locks are required |
| `COMPILER_INDEX_STORE_ENABLE=NO` | Disables indexing of AST data that won't be reused | Saves 20–30% runner CPU time and speeds up CI builds |
| `-parallel-testing-enabled YES` | Enables concurrent test runner workers on simulator images | Use on multi-core Mac runners to reduce test execution duration |
| `-maximum-concurrent-test-device-destinations` | Caps simulator instances to avoid out-of-memory kernel panics | Recommended `2` on standard GitHub Actions `macos-14` runner |
| `-only-testing:<Target>/<Suite>` | Runs a specific sub-suite instead of the full test target | Use for fast commit-stage verification loops |
| `-skip-testing:<Target>/<Suite>` | Skips known flaky or hardware-dependent test suites | Use for temporary quarantine while investigating regressions |

---

## 9. xcodebuild Exit Codes Diagnostic Matrix

| Exit Code | Common Root Cause | Surgical Remediation Command |
| :---: | :--- | :--- |
| **65** | Build failed (compilation error, code signing failure, or missing scheme) | Check `project.pbxproj` format version, verify scheme in `xcshareddata`, or pass `CODE_SIGNING_ALLOWED=NO` |
| **66** | Invalid target or scheme specified | Run `xcodebuild -list` to inspect available shared targets and schemes |
| **70** | Simulator destination unreachable or runtime crashed | Run `xcrun simctl list devices` to verify runtime availability; reboot simulator via `xcrun simctl shutdown all` |
| **74** | Missing or corrupted developer directory | Run `sudo xcode-select -s /Applications/Xcode_<version>.app/Contents/Developer` |

---

## 10. Operational CI Health Checklist

Before closing any Apple CI setup or maintenance task, verify:

- [ ] All `.xcscheme` files are committed under `xcshareddata/xcschemes/` (not in `xcuserdata/`).
- [ ] `objectVersion` in `project.pbxproj` is `<= 70` for compatibility with Xcode 16 runners.
- [ ] Headless signing bypass flags (`CODE_SIGNING_ALLOWED=NO`) are in place for PR workflows.
- [ ] Tests depending on Neural Engine or Metal hardware use `try XCTSkipIf()` when running under `CI=true`.
- [ ] GitHub Actions concurrency group is configured with `cancel-in-progress: true` to prevent runner saturation.

---

## 11. Production CI Automation Runbooks

### Runbook 1: Automated Test Failure Extraction via `xcresulttool`

When headless tests fail in CI, raw Xcode logs can exceed 50,000 lines. Use `xcresulttool` to extract exact test failures and assertion diagnostics into machine-readable JSON or markdown:

```bash
#!/usr/bin/env bash
set -euo pipefail

RESULT_BUNDLE="build/TestResults.xcresult"

if [ ! -d "$RESULT_BUNDLE" ]; then
  echo "❌ Error: Test result bundle not found at $RESULT_BUNDLE"
  exit 1
fi

echo "🔍 Parsing failures from $RESULT_BUNDLE..."

# Export test summaries as JSON
xcrun xcresulttool get --format json --path "$RESULT_BUNDLE" > test_summary.json

# Extract failure messages using jq
FAILURES=$(jq -r '
  .actions._values[]? |
  .actionResult.issues.testFailureSummaries._values[]? |
  "• \(.testCaseName._value): \(.message._value) (\(.documentLocationInCreatingWorkspace.url._value // "unknown location"))"
' test_summary.json)

if [ -n "$FAILURES" ]; then
  echo "🚨 Identified Test Failures:"
  echo "$FAILURES"
  # Write to GitHub Actions step summary if running in CI
  if [ -n "${GITHUB_STEP_SUMMARY:-}" ]; then
    echo "### ❌ Xcode Test Failures" >> "$GITHUB_STEP_SUMMARY"
    echo "$FAILURES" >> "$GITHUB_STEP_SUMMARY"
  fi
  exit 1
else
  echo "✅ All tests passed according to xcresult bundle."
fi
```

### Runbook 2: Optimized GitHub Actions Caching for SPM & DerivedData

Prevent SPM re-resolution and cold rebuilds across workflow runs with targeted cache keys:

```yaml
- name: Cache SPM Dependencies
  uses: actions/cache@v4
  with:
    path: |
      .build
      ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
    key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved', '**/*.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved') }}
    restore-keys: |
      ${{ runner.os }}-spm-

- name: Cache Build Artifacts
  uses: actions/cache@v4
  with:
    path: build/DerivedData
    key: ${{ runner.os }}-deriveddata-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-deriveddata-
```

### Runbook 3: Headless Ephemeral Keychain Management for Code Signing

When production signing is required on headless runners (e.g. for TestFlight or release builds), manage the temporary keychain lifecycle cleanly:

```bash
#!/usr/bin/env bash
set -euo pipefail

KEYCHAIN_NAME="ci-temp.keychain-db"
KEYCHAIN_PASSWORD=$(openssl rand -base64 32)

cleanup() {
  echo "🧹 Cleaning up ephemeral keychain..."
  security delete-keychain "$KEYCHAIN_NAME" 2>/dev/null || true
}
trap cleanup EXIT INT TERM

# 1. Create temporary keychain
security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_NAME"
security set-keychain-settings -lut 21600 "$KEYCHAIN_NAME"
security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_NAME"

# 2. Add to search list
security list-keychains -d user -s "$KEYCHAIN_NAME" $(security list-keychains -d user | tr -d '"')

# 3. Import certificates (base64-encoded secret from env)
echo "$APPLE_CERTIFICATE_BASE64" | base64 --decode > /tmp/cert.p12
security import /tmp/cert.p12 -k "$KEYCHAIN_NAME" -P "$APPLE_CERT_PASSWORD" -T /usr/bin/codesign -T /usr/bin/xcodebuild
rm -f /tmp/cert.p12

# 4. Partition list authorization for codesign
security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PASSWORD" "$KEYCHAIN_NAME"

echo "🔐 Ephemeral keychain ready for signed build execution."
```

### Runbook 4: Simulator Pre-Warming & Parallel Test Sharding

```bash
# Pre-warm target simulator before test execution
DEVICE_NAME="iPhone 16"
OS_VERSION="iOS-18-0"

UDID=$(xcrun simctl list devices available | grep "$DEVICE_NAME" | grep -v "unavailable" | head -n 1 | grep -o -E '[0-9A-F-]{36}')

if [ -z "$UDID" ]; then
  echo "⚠️ Target simulator not found, creating new instance..."
  UDID=$(xcrun simctl create "$DEVICE_NAME" "com.apple.CoreSimulator.SimDeviceType.iPhone-16" "com.apple.CoreSimulator.SimRuntime.$OS_VERSION")
fi

echo "🚀 Booting simulator $UDID..."
xcrun simctl boot "$UDID" || true
xcrun simctl bootstatus "$UDID" -b
```