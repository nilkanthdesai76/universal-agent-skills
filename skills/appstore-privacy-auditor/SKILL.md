---
name: appstore-privacy-auditor
description: >-
  Operational protocol for conducting comprehensive App Store submission pre-flight compliance audits. Use when: preparing an iOS/macOS/watchOS app for App Store submission, resolving rejection notice ITMS-91053, scanning codebases for Apple Required Reason APIs (UserDefaults, file modification timestamps, system boot time, disk space), generating or updating PrivacyInfo.xcprivacy manifests with valid reason codes, auditing App Tracking Transparency (ATT) and NSUserTrackingUsageDescription, or sanitizing release entitlements.
---

# App Store Privacy & Compliance Auditor 🛡️📋

The definitive operational manual for AI coding agents tasked with auditing iOS, macOS, watchOS, and tvOS apps for Apple App Store review compliance, privacy manifests, and zero-rejection submissions.

---

## 1. Executive Summary & Core Philosophy

Apple strictly enforces **Privacy Manifests (`PrivacyInfo.xcprivacy`)** for all applications and third-party frameworks invoking designated **Required Reason APIs**. Ingest engines in App Store Connect automatically reject submissions that link against these APIs without declaring approved reason codes.

1. **Failure Modes of AI Agents**:
   - Submitting apps without a `PrivacyInfo.xcprivacy` file bundled in target resources.
   - Calling APIs such as `UserDefaults`, file modification timestamps, system boot time, or disk space without declaring the approved Apple reason code.
   - Requesting `ATTrackingManager` tracking authorization without declaring `NSPrivacyTracking = true` and tracking domains in the privacy manifest.
   - Leaving test advertising IDs or non-sanitized staging URLs in production binaries.

2. **The Auditor's Mandate**:
   - **Zero Undeclared Required Reason APIs**: Every invocation of `UserDefaults`, `stat`, `sysctl`, or `volumeAvailableCapacity` must have an authorized reason code declared in `PrivacyInfo.xcprivacy`.
   - **Target Membership Verification**: Confirm `PrivacyInfo.xcprivacy` is included in the target's **Copy Bundle Resources** build phase.
   - **Tracking Transparency Consistency**: If IDFA is requested, declare `NSUserTrackingUsageDescription` in `Info.plist` and list tracking domains in `PrivacyInfo.xcprivacy`.

---

## 2. Official Apple Required Reason API Category & Reason Code Matrix

Apple mandates that if any binary (main app or dependency) links against APIs in the following categories, a corresponding `NSPrivacyAccessedAPITypes` entry must declare an approved reason code.

### 2.1 Category: User Defaults (`NSPrivacyAccessedAPITypeUserDefaults`)

Covered APIs: `UserDefaults`, `NSUserDefaults`, `CFPreferencesCopyAppValue`, `CFPreferencesSetAppValue`.

| Reason Code | Permitted Usage | Prohibited Usage |
|---|---|---|
| `CA92.1` | Accessing key-value pairs in the app's own user defaults database, or shared app group container for the same developer. | Tracking user activity across apps from different developers. |
| `1C8F.1` | Third-party SDK accessing key-value pairs solely to configure or persist SDK-internal state. | Reading host app preferences without authorization; cross-app fingerprinting. |
| `C56D.1` | Accessing user defaults across an app group consisting of apps distributed by the same developer. | Sharing defaults across different vendor team IDs. |

### 2.2 Category: File Timestamp (`NSPrivacyAccessedAPITypeFileTimestamp`)

Covered APIs: `stat`, `statfs`, `fstat`, `fstatat`, `getattrlist`, `getattrlistat`, `getattrlistbulk`, `futimes`, `utimes`, `NSURLContentModificationDateKey`, `NSURLCreationDateKey`, `FileManager.attributesOfItem(atPath:)`.

| Reason Code | Permitted Usage | Prohibited Usage |
|---|---|---|
| `C617.1` | Accessing timestamps of files inside the app's local container directories (Documents, Caches, Application Support). | Probing file timestamps in shared or temporary system paths to fingerprint device state. |
| `3B52.1` | Accessing timestamps of files explicitly selected by the user via `UIDocumentPickerViewController` or document interaction. | Accessing unselected external files without user intent. |
| `0A2A.1` | Third-party SDK accessing timestamps of files included in the SDK's own bundle or cache subfolder. | Probing host app file modification times. |
| `DDA9.1` | Displaying file creation/modification dates directly to the end user in the app UI. | Sending modification timestamps to remote telemetry servers for device identification. |

### 2.3 Category: System Boot Time (`NSPrivacyAccessedAPITypeSystemBootTime`)

Covered APIs: `systemUptime`, `mach_absolute_time()`, `clock_gettime(CLOCK_MONOTONIC_RAW)`.

| Reason Code | Permitted Usage | Prohibited Usage |
|---|---|---|
| `35F4.1` | Measuring relative time elapsed between user interactions or internal performance milestones within the running session. | Recording boot time to identify or track the device across reboot cycles. |
| `8FFB.1` | Calculating absolute timestamps for events that occurred while the app was running or backgrounded. | Exposing raw uptime to third-party ad networks. |

### 2.4 Category: Disk Space (`NSPrivacyAccessedAPITypeDiskSpace`)

Covered APIs: `statvfs`, `fstatvfs`, `NSURLVolumeAvailableCapacityKey`, `NSURLVolumeTotalCapacityKey`, `NSURLVolumeAvailableCapacityForImportantUsageKey`, `NSURLVolumeAvailableCapacityForOpportunisticUsageKey`.

| Reason Code | Permitted Usage | Prohibited Usage |
|---|---|---|
| `85F4.1` | Displaying available disk storage metrics directly to the user in settings or storage management UI. | Querying storage bytes to derive unique device hardware signatures. |
| `E174.1` | Verifying sufficient disk space exists before initiating a large file download, asset unpack, or media export. | Periodic storage polling as a background fingerprinting vector. |
| `7D9E.1` | Crash reporting, diagnostics, and debugging to determine whether low-memory or out-of-disk conditions triggered a fault. | Telemetry transmission when storage conditions are normal. |
| `B728.1` | Health check to manage local cache pruning and eviction policies. | Using remaining sector counts to differentiate devices. |

### 2.5 Category: Active Keyboards (`NSPrivacyAccessedAPITypeActiveKeyboards`)

Covered APIs: `UITextInputMode.activeInputModes`.

| Reason Code | Permitted Usage | Prohibited Usage |
|---|---|---|
| `3AC4.1` | Determining whether a custom third-party keyboard extension is installed or active to render custom input accessories. | Correlating keyboard language list combinations to fingerprint users. |
| `54DA.1` | Formatting or switching input text based on active system input languages. | Transmitting keyboard inventory to external analytics backends. |

---

## 3. Production `PrivacyInfo.xcprivacy` Template

The following production-ready XML schema illustrates complete compliance for an enterprise app utilizing `UserDefaults`, disk space checks, and crash analytics:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Tracking declaration: Set true only if data is linked with third-party data for targeted advertising -->
    <key>NSPrivacyTracking</key>
    <false/>

    <!-- Tracking domains: Must list all domains if NSPrivacyTracking is true -->
    <key>NSPrivacyTrackingDomains</key>
    <array/>

    <!-- Collected data types -->
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <!-- Crash Data Collection -->
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeCrashData</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
                <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            </array>
        </dict>
        <!-- Performance Diagnostics -->
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypePerformanceData</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>

    <!-- Required Reason APIs -->
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <!-- User Defaults Access -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
        <!-- File Timestamp Access -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>
            </array>
        </dict>
        <!-- System Boot Time -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeSystemBootTime</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>35F4.1</string>
            </array>
        </dict>
        <!-- Disk Space Inspection -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeDiskSpace</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>E174.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

---

## 4. Automated Static Binary & Source Scanner Script

Use this shell script in your CI/CD pipeline or pre-commit hook to detect unauthorized required reason API invocations:

```bash
#!/usr/bin/env bash
# audit-privacy-apis.sh — Audits compiled binary symbols and source references
set -euo pipefail

APP_PATH="${1:-}"

if [ -z "$APP_PATH" ]; then
  echo "Usage: $0 <path-to-.app-or-source-dir>"
  exit 1
fi

echo "🔍 Starting Apple Privacy Audit on: $APP_PATH"

FOUND_ISSUES=0

check_pattern() {
  local category="$1"
  local pattern="$2"
  local target="$3"
  
  echo -n "  Checking $category... "
  if [ -d "$target" ]; then
    # Source code search
    MATCHES=$(grep -rnE "$pattern" "$target" --include="*.swift" --include="*.m" --include="*.h" 2>/dev/null || true)
  else
    # Binary symbol search via nm / strings
    MATCHES=$(nm -u "$target" 2>/dev/null | grep -E "$pattern" || strings "$target" 2>/dev/null | grep -E "$pattern" || true)
  fi

  if [ -n "$MATCHES" ]; then
    echo "⚠️ DETECTED"
    echo "$MATCHES" | head -n 5 | sed 's/^/    /'
    FOUND_ISSUES=$((FOUND_ISSUES + 1))
  else
    echo "✅ Clean"
  fi
}

echo "=== Category Scans ==="
check_pattern "UserDefaults" "(UserDefaults|NSUserDefaults|CFPreferencesCopyAppValue)" "$APP_PATH"
check_pattern "FileTimestamp" "(NSURLContentModificationDateKey|NSURLCreationDateKey|attributesOfItemAtPath|stat64|fstatat)" "$APP_PATH"
check_pattern "SystemBootTime" "(systemUptime|mach_absolute_time|CLOCK_MONOTONIC_RAW)" "$APP_PATH"
check_pattern "DiskSpace" "(NSURLVolumeAvailableCapacityKey|NSURLVolumeTotalCapacityKey|statvfs)" "$APP_PATH"
check_pattern "ActiveKeyboards" "(activeInputModes)" "$APP_PATH"

echo "=== PrivacyInfo.xcprivacy Verification ==="
if [ -d "$APP_PATH" ]; then
  PRIVACY_FILE=$(find "$APP_PATH" -name "PrivacyInfo.xcprivacy" | head -n 1)
  if [ -z "$PRIVACY_FILE" ]; then
    echo "❌ ERROR: No PrivacyInfo.xcprivacy found in target directory!"
    FOUND_ISSUES=$((FOUND_ISSUES + 1))
  else
    echo "✅ Found $PRIVACY_FILE. Validating XML syntax..."
    plutil -lint "$PRIVACY_FILE"
  fi
fi

if [ "$FOUND_ISSUES" -gt 0 ]; then
  echo "⚠️ Audit completed with $FOUND_ISSUES findings. Ensure reasons are documented in PrivacyInfo.xcprivacy."
  exit 1
else
  echo "🎉 Privacy audit passed with zero undeclared findings."
  exit 0
fi
```

---

## 5. Detailed Submission Case Studies & Remedies

### Case Study 01: ITMS-91053 Missing UserDefaults Declaration (`CA92.1`)

#### Scenario Background
An iOS utility application saving user interface preferences via `UserDefaults.standard` was flagged during automated App Store Connect ingest with rejection notice:
```
ITMS-91053: Missing API declaration in Privacy Manifest - Your app’s code in the target 'MyApp' references one or more APIs that require reasons, including NSPrivacyAccessedAPITypeUserDefaults.
```

#### Problem Analysis
1. Binary symbol analysis via `nm` confirmed calls to `UserDefaults.standard.set(_:forKey:)`.
2. The target lacked a `PrivacyInfo.xcprivacy` manifest, or the file was not included in the "Copy Bundle Resources" build phase.

#### Resolution
1. Created `Resources/PrivacyInfo.xcprivacy`.
2. Added `NSPrivacyAccessedAPITypeUserDefaults` with authorized reason code `CA92.1` (accessing app's own user defaults).
3. Ensured the privacy manifest is checked under target membership in the Xcode project inspector.
4. Resubmitted archive to App Store Connect; ingest passed automatically.

---

### Case Study 02: ITMS-91053 Third-Party Framework File Timestamps (`0A2A.1`)

#### Scenario Background
An enterprise e-commerce app utilizing an embedded binary analytics framework failed validation:
```
ITMS-91053: Missing API declaration in Privacy Manifest - The binary 'Frameworks/VendorAnalytics.framework' references NSPrivacyAccessedAPITypeFileTimestamp without an authorized reason code.
```

#### Problem Analysis
1. The third-party framework queried bundle asset creation timestamps via `stat64` or `NSURLContentModificationDateKey` to verify cached offline assets.
2. The vendor had not yet released an updated framework containing a bundled `PrivacyInfo.xcprivacy`.

#### Resolution
1. Declared the third-party framework's required reason in the host application's top-level `PrivacyInfo.xcprivacy` as an interim mitigation:
   - Type: `NSPrivacyAccessedAPITypeFileTimestamp`
   - Reason: `0A2A.1` (third-party SDK accessing timestamps of files included in the SDK's own bundle).
2. Contacted the vendor to procure the updated `.xcframework` with its own embedded manifest.

---

### Case Study 03: ITMS-91054 Invalid Reason Code Suffix Formatting

#### Scenario Background
A development team received an immediate rejection after uploading an archive:
```
ITMS-91054: Invalid Reason Code - In NSPrivacyAccessedAPITypeFileTimestamp, 'C617' is not a recognized reason code.
```

#### Problem Analysis
The developer read Apple's human-readable documentation table and copied `C617` instead of the fully-qualified versioned identifier `C617.1`. Apple strictly enforces the `.1` version suffix.

#### Resolution
Updated `PrivacyInfo.xcprivacy`:
```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPITypeFileTimestamp</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>C617.1</string>
    </array>
</dict>
```

---

### Case Study 04: ITMS-91055 Disk Space Verification Before Asset Unpacking (`E174.1`)

#### Scenario Background
A photo and video editor was rejected during manual review when the reviewer requested clarification regarding `NSPrivacyAccessedAPITypeDiskSpace`.

#### Problem Analysis
1. The app invoked `NSURLVolumeAvailableCapacityKey` to ensure at least 500MB of free space was available before expanding video archives.
2. The privacy manifest declared disk space access, but the App Store reviewer flagged a mismatch between the declared reason code (`85F4.1` - display disk space to user) and the actual application UI (which did not display a disk space gauge).

#### Resolution
1. Corrected the reason code to `E174.1` (verifying sufficient disk space before creating or unpacking files).
2. Added an explanatory note in App Review Information referencing the unzipping function in `AssetManager.swift`.
3. Reviewer approved the submission on the subsequent pass.

---

## 6. SPM Resource Bundling Configuration

When creating reusable Swift packages that access required reason APIs, the `PrivacyInfo.xcprivacy` must be explicitly declared as a resource in `Package.swift`:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "SharedCore",
    platforms: [.iOS(.v17), .macOS(.v14)],
    products: [
        .library(name: "SharedCore", targets: ["SharedCore"]),
    ],
    targets: [
        .target(
            name: "SharedCore",
            resources: [
                .process("PrivacyInfo.xcprivacy")
            ]
        ),
        .testTarget(
            name: "SharedCoreTests",
            dependencies: ["SharedCore"]
        ),
    ]
)
```

> **Note**: Always use `.process("PrivacyInfo.xcprivacy")`. Xcode merges processed privacy manifests into the client application's final App Store submission bundle report automatically.

---

## 7. App Store Submission Rejection Triage (`ITMS-91053` to `91055`)

| Error Code | Rejection Cause | Resolution Protocol |
|---|---|---|
| `ITMS-91053` | Missing API declaration in Privacy Manifest | Identify the API cited in the Apple notification email (e.g. `NSPrivacyAccessedAPITypeUserDefaults`). Add the category to `NSPrivacyAccessedAPITypes` with the appropriate reason code (e.g. `CA92.1`). |
| `ITMS-91054` | Invalid Reason Code for declared API | Verify the reason string against Apple's authorized table. Common mistake: using `C617` instead of `C617.1` (the `.1` suffix is mandatory). |
| `ITMS-91055` | Undeclared third-party SDK manifest | Update third-party dependency via SPM/CocoaPods to a version that includes its own bundled `PrivacyInfo.xcprivacy`. If the SDK is abandoned, wrap calls or replace the library. |