---
name: iphone-duo-migrator
description: >-
  Operational protocol for adapting iOS and SwiftUI applications to iPhone Duo's foldable, dual-display architecture. Use when: modernizing apps for foldable devices (iPhone Duo), avoiding content clipping across the fold/hinge crease, replacing deprecated UIScreen.main with scene geometry and size classes, adopting the iOS 27.1 ReservedRegion / ArrangementView APIs, configuring vertical toolbars/tab bars, handling hinge angle interactions via onHingeChange, or supporting split-view multitasking and multiple scenes.
---

# iPhone Duo Migrator (Foldable & Dual-Display Adaptation) 📱📖

The definitive operational manual for AI coding agents tasked with retrofitting, architecting, and verifying iOS and SwiftUI applications for Apple iPhone Duo — Apple's first foldable iPhone (5.4″ outer display, 7.6″ inner display, launches on iOS 27.1).

All guidance below is sourced from Apple's official iPhone Duo Tech Talks and developer documentation (September 2026). **Never invent APIs or geometry constants** — only use the APIs documented here.

---

## 1. Executive Summary & Core Philosophy

iPhone Duo has two displays: an outer display that behaves like a standard iPhone, and an inner display that unfolds to a larger canvas where the fold/hinge, the FaceTime cameras, and side-aligned system bars shape the available space. Apps run without recompiling, but **how much of the inner display an app can use depends on which SDK it was built against** — layout is driven by *intent* (size classes, regions), never by *screen identity*.

1. **Failure Modes of AI Agents**:
   - Placing buttons, faces, or text across the fold crease instead of using Reserved Region queries.
   - Relying on `UIScreen.main` or `supportedInterfaceOrientations` — the inner display is `.regular` in **both** dimensions and does **not** honor interface orientations.
   - Inventing hinge gutters / posture thresholds instead of querying `reservedRegions(kind: .division)` and the hinge context APIs.
   - Building custom `UIToolbar`/`UINavigationBar`/`UITabBar` instances, whose content is **not considered** for the vertical shared bar.
   - Not testing in Split View, where every app participates in side-by-side multitasking.

2. **The Migrator's Mandate**:
   - **Intent-Driven Layout**: Use horizontal/vertical size classes, standard containers, and safe areas — most adaptation is free.
   - **Fold Awareness**: Query reserved regions (`division` = fold, `occlusion` = camera); never hardcode crease metrics.
   - **Scene-Relative Geometry**: Access screens via the window scene, traits, and `GeometryReader`.
   - **Hinge for Effects, Regions for Layout**: Hinge angle drives interactions/effects only; layout decisions come from arrangement and region APIs.

---

## 2. Device Facts & SDK Tier Matrix

```
+-------------------------------------------------------------------------------------------+
|                          IPHONE DUO DEVICE & SDK FACTS                                    |
+------------------+------------------------------------------------------------------------+
| Displays         | Outer 5.4" (compact width — behaves like standard iPhone)              |
|                  | Inner 7.6" (regular width AND height — room for sidebars)              |
| Fold             | Backed by a "division" reserved region; active only when folded,       |
|                  | zero width when flat. NEVER hardcode a crease dimension.               |
| Cameras          | Backed by "occlusion" reserved regions (FaceTime camera on displays)   |
| Size classes     | Inner display is regular in BOTH dimensions; does NOT honor            |
|                  | supportedInterfaceOrientations. Use size classes, NOT orientation.     |
| Split View       | All apps participate; two apps side by side; video+app stacking layout |
| Multitasking     | (outer display landscape overflows bars most)                          |
| Multiple windows | First iPhone with multiple scene instances; NEW windows can only be    |
|                  | created on the INNER display (outer is reserved). Handle errors when   |
|                  | requesting scenes; UIWindowSceneActivation hides itself when unavailable|
+------------------+------------------------------------------------------------------------+
| SDK Tier         | Behavior on iPhone Duo                                                 |
+------------------+------------------------------------------------------------------------+
| Built with       | App runs unchanged; content centered in middle of inner display        |
| older SDK        |                                                                        |
| iOS 27 SDK       | App extends left of the status bar on the inner display                |
| iOS 27.1 SDK     | App reaches the screen edge; standard navigation/toolbar/tab-bar       |
| (Xcode 27.1)     | items lay out VERTICALLY in the shared side bar; ReservedRegion,       |
|                  | ArrangementView, onHingeChange, scene accessories available            |
+------------------+------------------------------------------------------------------------+
```

**Tooling**: Xcode 27.1. Simulate every pose in **DeviceHub** — on-screen controls let you open, close, rotate, and fold the device. In Xcode 27.1 the old app-modernization skill is now **App Resizability** (supports SwiftUI and iPhone Duo).

---

## 3. Geometry Migration Protocol

### 3.1 Eliminating Deprecated `UIScreen.main` References

On a two-display device, the "main screen" is ambiguous and will be deprecated. Access screens dynamically from the window scene; read display scale from the trait collection.

```swift
// ❌ WRONG: ambiguous on a two-display device
let screen = UIScreen.main
let screenScale = UIScreen.main.scale
let screenWidth = UIScreen.main.bounds.width

// ✅ CORRECT: access the screen from the window scene
let screen = window?.windowScene?.screen

// ✅ CORRECT: read scale from traits instead of the screen singleton
let screenScale = traitCollection.displayScale

// ✅ CORRECT (SwiftUI): read from the environment / geometry
GeometryReader { proxy in
    let width = proxy.size.width
    let safe = proxy.safeAreaInsets   // may be ASYMMETRIC — handle each side independently
}

// ✅ CORRECT (SwiftUI size classes — the primary layout signal)
@Environment(\.horizontalSizeClass) private var hSizeClass
@Environment(\.verticalSizeClass)   private var vSizeClass
```

### 3.2 Matching Screen Corners — Concentricity (iOS 26+)

```swift
// SwiftUI: content fits the screen's actual rounded corners
ConcentricRectangle()
    .fill(Color.green)
    .padding(8.0)
    .ignoresSafeArea()

// UIKit: equivalent via UICornerConfiguration
```

---

## 4. Safe Areas & Layout Margins (Asymmetric!)

Safe areas and layout margins are **often asymmetric** on iPhone Duo. Handle each side independently; never assume opposite insets are equal.

```swift
// ❌ WRONG: assumes symmetric insets
let width = view.bounds.width - view.safeAreaInsets.left * 2

// ✅ CORRECT: handle each side independently
let width = view.bounds.inset(by: view.safeAreaInsets).width

// ✅ CORRECT: foreground interactive content stays inside the safe area
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)

// ✅ CORRECT: background artwork extends past it
backgroundView.frame = view.bounds        // UIKit
.background(...).ignoresSafeArea()         // SwiftUI
```

Standard bars lay out outside the safe area and automatically avoid the status bar and camera. **Test in Split View** — multitasking changes both size classes and safe areas.

---

## 5. Reserved Regions — The Fold-Aware Layout API (iOS 27.1)

Reserved regions are hardware-shaped spaces the layout must adapt to: the **fold** (a `.division` region — it divides a larger area into smaller ones) and the **cameras** (`.occlusion` regions — they occlude rather than divide). Treat them like any other adaptable area (e.g., iPad window controls).

- Regions are **active or inactive**. Only active regions are returned by default; the fold's division region is active only when folded and has **zero width when flat**. Use `.includeInactive` for high-level decisions (e.g., preferring an even number of grid columns around the crease).
- The system automatically repositions alerts, action sheets, menus, and popovers around reserved regions — prefer system containers over custom presentations.

```swift
// SwiftUI: query via GeometryProxy (GeometryReader or onGeometryChange)
GeometryReader { proxy in
    let fold = proxy.reservedRegions(kind: .division)                    // fold crease
    let foldAll = proxy.reservedRegions(kind: .division,
                                         options: .includeInactive)
    let cameras = proxy.reservedRegions(kind: .occlusion)               // FaceTime camera
    let frames = fold.map(\.frame)
    // Incorporate frames into your own layout — never hardcode gutter sizes
}

// UIKit: query on UIView
let regions = view.reservedRegions(kind: .division)
let frames = regions.map(\.frame)
```

### 5.1 Displacement Patterns

When content can't flow around a region, **displace** it (adjust the frame of existing elements):

- Move elements independently when they adapt alone; together when they work as a unit. Avoid excessive movement that weakens visual relationships.
- Book-style partial fold → alerts move to the **trailing** side (near where they appear as the device closes). Tabletop → top region for viewed-at-a-distance content, bottom region for interactive controls.
- Scrolling content (articles, feeds) must NOT displace — it flows continuously.

### 5.2 Claiming Space — ReservedRegion / UIViewReservedRegion (iOS 27.1)

New in iOS 27.1, `ReservedRegion` (SwiftUI) and `UIViewReservedRegion` (UIKit) let custom UI claim as much screen space as possible without colliding with system UI — useful for custom bars and edge-to-edge designs.

---

## 6. ArrangementView — Split & Overlay Two-View Layouts (iOS 27.1)

An **arrangement** is a rule that maps inputs (size classes, aspect ratio, active division regions) to outputs (which view shows, what frame it gets). `ArrangementView` takes a **primary** and **secondary** view and sits inside a `NavigationStack`.

```swift
// SwiftUI
NavigationStack {
    ArrangementView {
        PlayerView()        // primary
    } secondary: {
        UpNextView()        // secondary
    }
    .arrangementViewStyle(.split)                       // default
    .arrangementViewStyle(.split.axes(.horizontal))     // restrict to one axis
    // .arrangementViewStyle(.overlay)                  // foreground/background
}
```

```swift
// UIKit
let arrangementVC = UIArrangementViewController()
let navController = UINavigationController(rootViewController: arrangementVC)
arrangementVC.setViewController(PlayerViewController(), for: .primary)
arrangementVC.setViewController(UpNextViewController(), for: .secondary)
arrangementVC.updateArrangement(.split.axes(.horizontal))   // UISplitArrangement
```

Key behaviors:

- **Split** (default): divides bounds between the two views — horizontally when wider than tall, vertically when taller. If it can't split along its axis, it shows only one view.
- **Overlay**: prefers above/below; moves to side-by-side when the device folds. Read `\.overlayArrangementZIndex` (SwiftUI env) or `arrangementVC.state(for: .primary)?.zIndex` (UIKit) to switch between collapsed/expanded variants as the user folds.
- **Choosing**: HStack/VStack layouts → split; ZStack layouts → overlay. Overlay when there's a clear foreground/background relationship; split for main/detail where neither view should be obscured.

**When NOT to use an ArrangementView**: it provides no navigation infrastructure — don't put `NavigationSplitView` inside one; and don't put one inside a `List`/`ScrollView` (scrollable container).

---

## 7. Hinge Interactions — Effects Only, Not Layout (iOS 27.1)

SwiftUI's `onHingeChange` modifier and UIKit's `UIHingeInteraction` report hinge status (`.closed` / `.partiallyOpen` / `.fullyOpen`) plus continuous **hinge angle**. Use for interactive effects (pitch bends, game controls); **never** for layout — use Reserved Regions and Arrangements for layout.

```swift
struct InstrumentView: View {
    @State private var pitchBend: Double = 0   // 0 = no bend, 1 = deepest

    var body: some View {
        GuitarView(pitchBend: pitchBend)
            .onHingeChange { _, context in
                // A null hinge means the device doesn't have one
                if let hinge = context.hinge, hinge.status == .partiallyOpen {
                    pitchBend = calculatePitchBend(angle: hinge.angle)
                } else {
                    pitchBend = 0   // reset when the angle isn't being read
                }
            }
    }
}
```

---

## 8. Navigation, Standard Containers & Sidebars

`NavigationStack`, `NavigationSplitView`, `TabView`, `UISplitViewController`, and `UITabBarController` are **fully adaptive across every pose** — columns collapse when closed and tile or overlay when open. Sheets, popovers, context menus, and alerts adapt too. Prefer these over custom containers.

On the inner display you can opt into a **sidebar** placement for richer navigation:

```swift
// SwiftUI
TabView { ... }
    .defaultTabBarPlacement(.sidebar)

// UIKit
tabBarController.sidebar.preferredPlacement = .sidebar
```

---

## 9. Vertical Bars — The Shared Bar Region (Raise the Bar, iOS 27.1)

On the inner display, navigation/toolbar/tab-bar controls move into a **shared vertical bar region** on the side (hardware-aligned — same side in RTL). Requirements and rules:

- **Opt in automatically** by rebuilding with the iOS 27.1 SDK and using bars from navigation containers. Content from custom `UIToolbar`/`UINavigationBar`/`UITabBar` instances is **not considered** — prefer `UINavigationController`/`UITabBarController`.
- In split views, **only the detail column** participates (inspectors get no bar). The bar returns to horizontal in portrait and on the outer display.
- Ordering: top = back/close (`placement: .cancellationAction`; UIKit `navigationItem.leftItemsSupplementBackButton = false` + leading item groups), then prominent actions (`.topBarPinnedTrailing` / `pinnedTrailingGroup`).

```swift
// Toolbar content (SwiftUI) — pair toolbar with NavigationStack/NavigationSplitView
NavigationStack {
    ContentView()
        .toolbar {
            ToolbarItem(placement: .bottomBar) { ... }
            ToolbarItem(placement: .topBarPinnedTrailing) { ... }
        }
}
```

- **Item axis behavior** (`axisBehavior` in UIKit, `.axisBehavior(...)` on ToolbarItem): icon items go vertical by default; text-only items stay horizontal. Keep transitioning items (symbol ↔ text) on `.horizontalOnly`; set `.verticalPreferred` only when a custom view truly supports a vertical representation. Prefer symbol-only items (use `.badge(n)` — iOS 26 — to turn a text+symbol item into symbol-only).
- **Custom views**: must fit the bar's fixed width or adapt vertically. Read the bar edge: `\.toolbarVerticalEdge` (SwiftUI env) / `traitCollection.verticalBarEdge` (UIKit). Flexible spacers are zero-size vertically; vertical bars have no scroll-edge effect by default.
- **Overflow**: items overflow bottom-to-top. Toolbars compress before the tab bar by default; control with `toolbarVerticalCompressionBehavior(.prefersToolbarItems)` (SwiftUI) / `navigationItem.verticalBarCompressionBehavior = .prefersBarItems` (UIKit). Consolidate custom overflow into the system menu via `ToolbarOverflowMenu` / `additionalOverflowItems` (UIDeferredMenuElement); reserve the ellipsis for overflow only.
- **Visibility priority**: `.visibilityPriority(.high/.low/...)` (SwiftUI) / `item.visibilityPriority` (UIKit) — frequently used actions and status-conveying (badged) items should overflow last.
- **Opt out** when inappropriate (e.g., single-page bottom-heavy apps like Calculator, or one-control sheets):

```swift
// SwiftUI
ContentView().toolbarVerticalBehavior(.disabled)

// UIKit
override var preferredVerticalBarBehavior: UIVerticalBarBehavior { .disabled }
```

---

## 10. Multiple Displays, Scenes & Accessories

- **Split View multitasking**: every app participates (two apps side by side; a new video+app stacking layout exists). If your app resizes on iPad / iPhone Mirroring on Mac, you're most of the way there.
- **Multiple scenes**: iPhone Duo is the first iPhone with multiple app instances. New windows can only be created on the **inner** display (outer is reserved). Handle errors when requesting scenes; use `UIWindowSceneActivation` action, which hides itself automatically when new windows aren't available.

### 10.1 Scene Accessories — Content on Both Displays

Scene accessories pair supplementary content with your main UI (like a controller on one display and content on another). Availability is system-controlled (enabled by default, toggled anytime) — respond with `onAvailabilityChange`.

```swift
// Camera capture accessory: pairs UI on the outer display while main UI
// stays on the inner display (e.g., a teleprompter the subject can see).
// Available when full screen on inner display + active camera session;
// register on the same view as your camera UI.
struct CameraRootView: View {
    @State private var model = TeleprompterModel()

    var body: some View {
        CameraView(model: model)
            .sceneAccessory {
                CameraCaptureAccessory(isEnabled: $model.isEnabled) {
                    TeleprompterView(model: model)
                }
                .onAvailabilityChange { newValue in
                    model.isAvailable = newValue   // e.g., device closed → disable toggle
                }
            }
            .toolbar {
                TeleprompterToggle(isEnabled: $model.isEnabled)
                    .disabled(!model.isAvailable)
            }
    }
}
```

Camera apps should additionally review Apple's camera guidance for iPhone Duo: virtual vs. individual front cameras, choosing cameras by the direction they face, preview layout/aspect ratio, and rotation APIs (Tech Talk 111465 — *Build a great camera experience for iPhone Duo*).

---

## 11. iPhone Duo Deployment Verification Checklist

Before certifying an app for iPhone Duo:

- [ ] **Built with iOS 27.1 SDK / Xcode 27.1** — app reaches the screen edge on the inner display; vertical shared bar active.
- [ ] **No `UIScreen.main` references** — screens read from `window?.windowScene?.screen`; scale from `traitCollection.displayScale`.
- [ ] **Size classes, not orientation** — layout decisions use `horizontalSizeClass`/`verticalSizeClass` (inner display is regular × regular and ignores `supportedInterfaceOrientations`).
- [ ] **Asymmetric safe areas** — insets handled per side via `bounds.inset(by: safeAreaInsets)`; foreground content inside safe area, backgrounds extend past it.
- [ ] **Fold-aware content** — nothing critical spans the crease; custom layouts query `reservedRegions(kind: .division)`; `.occlusion` regions respected for cameras; no hardcoded crease dimensions.
- [ ] **Standard navigation containers** — `NavigationSplitView`/`TabView` (or UIKit equivalents) used; sidebar placement opted into where richer nav fits; no custom bar instances blocking vertical bar participation.
- [ ] **Vertical bar audit** — symbol-only items preferred, titles always provided, axis behaviors set on custom views, badges used, overflow consolidated into system menu, visibility priorities assigned, edge read via `toolbarVerticalEdge`.
- [ ] **Hinge used only for effects** — `onHingeChange`/`UIHingeInteraction` drive interactions; layout comes from Reserved Regions/Arrangements.
- [ ] **Split View multitasking tested** — app resizes cleanly side-by-side; tested in every pose via DeviceHub (open, close, rotate, fold).
- [ ] **Scene support** — new-scene requests handle errors; `UIWindowSceneActivation` used; no new windows on outer display; accessories respond to `onAvailabilityChange`.
- [ ] **Single-screen parity** — app still behaves flawlessly on all other iPhones and iPads.

---

## 12. Official Apple References

| Resource | URL |
|---|---|
| iPhone Duo developer hub | https://developer.apple.com/iphone-duo/ |
| Tech Talk 111461 — Prepare your app for iPhone Duo (minimum work: SDK, size classes, safe areas, reserved regions) | https://developer.apple.com/videos/play/tech-talks/111461/ |
| Tech Talk 111462 — Raise the bar with iPhone Duo (vertical bars, overflow, axis behavior) | https://developer.apple.com/videos/play/tech-talks/111462/ |
| Tech Talk 111463 — Strike a pose with adaptive layouts on iPhone Duo (reserved regions, displacement, ArrangementView) | https://developer.apple.com/videos/play/tech-talks/111463/ |
| Tech Talk 111464 — Leverage multiple displays and scenes on iPhone Duo (hinge, multi-scene, scene accessories) | https://developer.apple.com/videos/play/tech-talks/111464/ |
| Tech Talk 111465 — Build a great camera experience for iPhone Duo | https://developer.apple.com/videos/play/tech-talks/111465/ |
| Tech Talk 111466 — Design for iPhone Duo (design principles, poses, side controls) | https://developer.apple.com/videos/play/tech-talks/111466/ |
| HIG — Designing for iPhone Duo | https://developer.apple.com/design/ |
| Docs — Preparing your app for iPhone Duo | https://developer.apple.com/documentation/ |
