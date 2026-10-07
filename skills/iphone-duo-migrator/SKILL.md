---
name: iphone-duo-migrator
description: >-
  Operational protocol for adapting iOS and SwiftUI applications to foldable and dual-screen hardware architectures. Use when: modernizing apps for foldable devices (iPhone Duo), preventing UI elements from being occluded by the physical center hinge, implementing dynamic posture state machines (flat, half-folded, dual portrait, dual landscape), replacing deprecated UIScreen.main.bounds with scene geometry, or ensuring graceful single-screen fallbacks.
---

# iPhone Duo Migrator (Dual-Screen & Foldable Adaptation) 📱📖

The definitive operational manual for AI coding agents tasked with retrofitting, architecting, and verifying iOS and SwiftUI applications for dual-screen and foldable device hardware.

---

## 1. Executive Summary & Core Philosophy

Foldable and dual-screen hardware introduces physical discontinuities (such as the center hinge crease) and dynamic postures (flat, tabletop half-fold, dual portrait book mode, dual landscape). Apps designed for static single screens fail catastrophically when text or buttons are split across the physical hinge.

1. **Failure Modes of AI Agents**:
   - Centering critical buttons, faces, or text directly across the physical hinge crease.
   - Relying on deprecated `UIScreen.main.bounds` rather than dynamic window scene geometry.
   - Failing to provide graceful fallbacks on standard single-screen iPhones and iPads.
   - Ignoring orientation changes and posture transitions, leading to clipped views or awkward aspect ratios.

2. **The Migrator's Mandate**:
   - **Hinge Occlusion Avoidance**: Treat the hinge region (typically 12–16 pt) as non-renderable screen space.
   - **Posture-Aware Layouts**: Adapt layouts dynamically according to device posture (`.flat`, `.halfFolded`, `.dualPortrait`, `.dualLandscape`).
   - **Scene-Relative Geometry**: Use `GeometryReader` and `UIWindowScene` metrics exclusively.
   - **Adaptive Single-Screen Fallback**: Fluidly collapse into standard `NavigationSplitView` or vertical `VStack` on single-screen devices.

---

## 2. Foldable Hardware Geometry & Posture Matrix

```
+-----------------------------------------------------------------------------------------+
|                               DUAL-SCREEN POSTURE TAXONOMY                              |
+----------------+-------------+----------------------------------------------------------+
| Posture State  | Hinge Angle | Screen Allocation & Layout Strategy                      |
+----------------+-------------+----------------------------------------------------------+
| .flat          | 180 degrees | Unified wide canvas with hinge avoidance margin          |
| .halfFolded    | 80-110 deg  | Tabletop: Top screen display, bottom screen controls     |
| .dualPortrait  | 130-180 deg | Book mode: Left screen master list, right screen detail  |
| .dualLandscape | 130-180 deg | Stacked mode: Top canvas, bottom inspector & timeline    |
+----------------+-------------+----------------------------------------------------------+
```

---

## 3. Production SwiftUI Foldable Architecture

### 3.1 Device Posture State Machine (`DevicePosture.swift`)

```swift
import SwiftUI

public enum DevicePosture: Sendable, Equatable {
    case singleScreen
    case flat
    case halfFolded(angle: Double)
    case dualPortrait
    case dualLandscape

    public var isDualScreen: Bool {
        switch self {
        case .singleScreen: return false
        default: return true
        }
    }

    public var isTabletop: Bool {
        if case .halfFolded = self { return true }
        return false
    }
}

@Observable
public final class PostureManager {
    public static let shared = PostureManager()

    public var currentPosture: DevicePosture = .singleScreen
    public var hingeRect: CGRect = .zero
    public var hingeGutter: CGFloat = 16.0

    private init() {
        // Observe scene changes or sensor updates
    }

    public func updatePosture(for windowScene: UIWindowScene?) {
        guard let windowScene else {
            currentPosture = .singleScreen
            return
        }

        let bounds = windowScene.screen.bounds
        let isWide = bounds.width > 700 && bounds.height > 600

        if isWide {
            // Screen geometry indicates dual-screen / unfolded state
            let hingeX = (bounds.width - hingeGutter) / 2.0
            hingeRect = CGRect(x: hingeX, y: 0, width: hingeGutter, height: bounds.height)
            currentPosture = .dualPortrait
        } else {
            hingeRect = .zero
            currentPosture = .singleScreen
        }
    }
}
```

---

### 3.2 Adaptive Two-Pane Layout Container (`TwoPaneView.swift`)

A flexible dual-pane container that automatically separates content across the hinge or collapses into a single-pane navigation flow:

```swift
import SwiftUI

public struct TwoPaneView<Primary: View, Secondary: View>: View {
    let primary: Primary
    let secondary: Secondary
    let primaryRatio: CGFloat
    @Environment(\.horizontalSizeClass) private var sizeClass

    public init(
        primaryRatio: CGFloat = 0.5,
        @ViewBuilder primary: () -> Primary,
        @ViewBuilder secondary: () -> Secondary
    ) {
        self.primaryRatio = primaryRatio
        self.primary = primary()
        self.secondary = secondary()
    }

    public var body: some View {
        GeometryReader { proxy in
            let isDual = proxy.size.width > 700
            let hingeWidth: CGFloat = isDual ? 16.0 : 0.0

            if isDual {
                HStack(spacing: hingeWidth) {
                    primary
                        .frame(width: (proxy.size.width - hingeWidth) * primaryRatio)
                    
                    // Physical hinge margin gap
                    Divider()
                        .opacity(0)
                        .frame(width: hingeWidth)

                    secondary
                        .frame(width: (proxy.size.width - hingeWidth) * (1.0 - primaryRatio))
                }
            } else {
                // Fallback for single-screen devices
                NavigationSplitView {
                    primary
                } detail: {
                    secondary
                }
            }
        }
    }
}
```

---

### 3.3 Tabletop Media Player Controller (`TabletopPlayerView.swift`)

In tabletop mode (half-folded on a flat surface), the upper half hosts video playback while the lower half hosts playback scrubbers and control dials:

```swift
import SwiftUI
import AVKit

public struct TabletopPlayerView: View {
    @State private var isPlaying: Bool = true
    @State private var progress: Double = 0.45

    public var body: some View {
        GeometryReader { proxy in
            let isTabletop = proxy.size.height > proxy.size.width

            VStack(spacing: 0) {
                // Upper Screen: Media Display
                ZStack {
                    Color.black
                    Text("📺 Video Playback Screen")
                        .foregroundStyle(.white)
                        .font(.title2)
                }
                .frame(height: proxy.size.height / 2)

                // Physical Hinge Crease Cushion
                Rectangle()
                    .fill(Color.gray.opacity(0.1))
                    .frame(height: 12)

                // Lower Screen: Control Deck
                VStack(spacing: 24) {
                    Slider(value: $progress)
                        .padding(.horizontal, 24)

                    HStack(spacing: 40) {
                        Button(action: { progress = max(0, progress - 0.1) }) {
                            Image(systemName: "gobackward.10")
                                .font(.system(size: 28))
                        }

                        Button(action: { isPlaying.toggle() }) {
                            Image(systemName: isPlaying ? "pause.circle.fill" : "play.circle.fill")
                                .font(.system(size: 54))
                        }

                        Button(action: { progress = min(1, progress + 0.1) }) {
                            Image(systemName: "goforward.10")
                                .font(.system(size: 28))
                        }
                    }
                }
                .frame(maxWidth: .infinity, maxHeight: .infinity)
                .background(Color(uiColor: .secondarySystemBackground))
            }
        }
        .ignoresSafeArea(.all, edges: .top)
    }
}
```

---

## 4. Modern Geometry Migration Protocol

### 4.1 Eliminating Deprecated `UIScreen.main` References

Modern iOS multi-window and foldable architectures break when referencing global screen singletons:

```swift
// ❌ WRONG: Static screen bounds assume single immutable display
let screenWidth = UIScreen.main.bounds.width
let screenHeight = UIScreen.main.bounds.height

// ✅ CORRECT: Query window scene geometry via GeometryReader
GeometryReader { proxy in
    let currentWidth = proxy.size.width
    let safeTop = proxy.safeAreaInsets.top
    let safeBottom = proxy.safeAreaInsets.bottom
    // Use container proxy metrics
}

// ✅ CORRECT (in UIKit): Query attached UIWindowScene
guard let windowScene = view.window?.windowScene else { return }
let sceneBounds = windowScene.screen.bounds
```

---

## 5. Foldable UI Adaptation Verification Checklist

Before certifying an application for foldable or dual-screen deployment:

- [ ] **No Hinge Occlusion**: Zero interactive controls, labels, or avatar images are positioned across the center hinge crease.
- [ ] **Dynamic Posture Support**: App transitions cleanly between `.singleScreen`, `.flat`, and `.halfFolded` without restarts.
- [ ] **Tabletop Mode Verified**: Media playback or camera viewfinders position viewfinders on the upper screen and controls on the lower screen.
- [ ] **Scene Bounds Used**: Deprecated `UIScreen.main.bounds` is eliminated across all views and view controllers.
- [ ] **Single-Screen Parity**: The app functions flawlessly on standard iPhone and iPad form factors using standard responsive layouts.