---
name: swiftui-layout-debugger
description: >-
  Operational protocol for diagnosing and repairing SwiftUI layout anomalies and runtime rendering bugs. Use when: debugging SwiftUI layout glitches, eliminating 'Modifying state during view update' console warnings, resolving infinite body execution cycles, fixing clipping, overflow, or unexpected Spacer expansion, complying with MacBook notch and iPhone Dynamic Island safe areas, or supporting Dynamic Type accessibility text scaling.
---

# SwiftUI Layout Debugger (Runtime Diagnostics & View Geometry) 🎨📐

The definitive operational manual for AI coding agents tasked with analyzing, debugging, and repairing SwiftUI layout anomalies, state update cycles, dynamic sizing, notch compliance, and view hierarchy performance.

---

## 1. Executive Summary & Core Philosophy

SwiftUI's declarative layout engine relies on a strict single-direction data flow: state produces views, and views produce layout geometry. When agents misunderstand this mental model, they introduce common critical defects:

1. **The Failure Modes of AI Agents**:
   - **State Mutation During Render Loops**: Writing to `@State` or `@Binding` inside the view's computed `body`, triggering `"Modifying state during view update"` warnings and runaway CPU spin loops.
   - **Safe-Area Clipping & Notch Invasions**: Unconditionally applying `.ignoresSafeArea()` and pushing interactive buttons behind MacBook notches or iPhone Dynamic Islands.
   - **Fixed-Frame Rigidity**: Hardcoding widths like `.frame(width: 375)` that break across varying device widths, iPads, or Split View multitasking.
   - **Truncated Text & Accessibility Disasters**: Disregarding Dynamic Type scaling, resulting in clipped labels and unusable interfaces for vision-impaired users.
   - **Touch Target Collapse**: Tiny button icons lacking `.contentShape(Rectangle())` with hit testing targets under 44×44 pt.

2. **The Debugger's Mandate**:
   - **Zero Synchronous State Mutations in View Bodies**: Views are pure projections of state. State modifications belong exclusively in action handlers (`Button(action:)`), `.onChange(of:)`, or asynchronous `.task` blocks.
   - **GeometryReader Containment**: Always constrain `GeometryReader` within explicit frames or background overlays; it expands greedily by default and pushes sibling views off-screen.
   - **Dynamic Type Scalability**: Never lock font sizes without `@ScaledMetric` or adaptive containers like `ViewThatFits`.
   - **Touch Target Accessibility**: Enforce minimum 44×44 pt hit testing bounds for every interactive control via `.frame(minWidth: 44, minHeight: 44)` and `.contentShape(Rectangle())`.

---

## 2. SwiftUI Layout Engine Mental Model

```
+-------------------------------------------------------------------------+
|                      SWIFTUI 3-STEP LAYOUT PROCESS                      |
+-------------------------------------------------------------------------+
| 1. PROPOSAL           | Parent proposes a size to the child.            |
|                       | -> (e.g. "I have 300x600 available, how much     |
|                       |    do you want?")                               |
+-----------------------+-------------------------------------------------+
| 2. DETERMINATION      | Child determines its own required size.         |
|                       | -> (e.g. Text responds: "I only need 120x30.")  |
+-----------------------+-------------------------------------------------+
| 3. PLACEMENT          | Parent places the child in its coordinate space.|
|                       | -> Centers by default; respects alignments.     |
+-------------------------------------------------------------------------+
```

---

## 3. Systematic Diagnostic Protocol

When a view flickers, clips, loops, or renders off-center:

```
[Step 1: Symptom Classification]
       │
       ├── Infinite Update Loop ──► Audit view body for @State / @Binding mutations
       ├── Clipping / Overflow  ──► Audit padding, spacing, and parent stack constraints
       ├── Safe-Area Intrusion  ──► Audit .ignoresSafeArea edges and window insets
       ├── Hit-Testing Failure  ──► Audit frame bounds and .contentShape() modifiers
       └── Frame Jitter / Lag   ──► Audit List/LazyVStack view identity in ForEach
```

### 3.1 Runtime Triage with `Self._printChanges()`

To identify which state property triggers unexpected re-renders, place `Self._printChanges()` inside the view's computed `body`:

```swift
struct ProfileCardView: View {
    @Bindable var viewModel: ProfileViewModel

    var body: some View {
        #if DEBUG
        let _ = Self._printChanges()
        #endif

        VStack(spacing: 12) {
            Text(viewModel.userName)
            // ...
        }
    }
}
```

Console output indicates exact trigger:
```
ProfileCardView: @Bindable var viewModel changed.
ProfileCardView: _isSelected changed.
```

---

## 4. Comprehensive Catalog of 12 Real SwiftUI Architectural Bugs & Fixes

### Bug 01: Modifying State During View Update
- **Console Warning**: `[SwiftUI] Modifying state during view update, this will cause undefined behavior.`
- **Root Cause**: Mutating `@State` or `@Binding` inside the computed `body` property.
- **The Fix**:
  ```swift
  // ❌ WRONG:
  var body: some View {
      let _ = { self.hasRendered = true }()
      Text("Hello")
  }

  // ✅ CORRECT:
  var body: some View {
      Text("Hello")
          .onAppear {
              hasRendered = true
          }
  }
  ```

---

### Bug 02: Greedily Expanding GeometryReader Ruining Parent Stack
- **Root Cause**: `GeometryReader` greedily expands to fill all proposed parent space, displacing siblings.
- **The Fix**:
  Use `GeometryReader` inside `.background()` to observe size without affecting parent layout:
  ```swift
  // ✅ CORRECT: Measure size passively via background preference
  struct SizePreferenceKey: PreferenceKey {
      static var defaultValue: CGSize = .zero
      static func reduce(value: inout CGSize, nextValue: () -> CGSize) {
          value = nextValue()
      }
  }

  Text("Title")
      .background(
          GeometryReader { proxy in
              Color.clear.preference(key: SizePreferenceKey.self, value: proxy.size)
          }
      )
      .onPreferenceChange(SizePreferenceKey.self) { size in
          measuredSize = size
      }
  ```

---

### Bug 03: Safe-Area Clipping & Notch Invasions
- **Root Cause**: Indiscriminate `.ignoresSafeArea()` pushes interactive buttons under the notch or status bar.
- **The Fix**:
  Use `.safeAreaInset(edge:)` to anchor persistent bars while respecting hardware safe zones:
  ```swift
  // ✅ CORRECT: Persistent bottom bar respecting home indicator
  ScrollView {
      VStack(spacing: 16) {
          ContentListView()
      }
  }
  .safeAreaInset(edge: .bottom) {
      CheckoutButton()
          .padding()
          .background(.ultraThinMaterial)
  }
  ```

---

### Bug 04: Truncated Text with Dynamic Type
- **Root Cause**: Hardcoded frames cause text to truncate when users increase text size in Accessibility settings.
- **The Fix**:
  Combine `@ScaledMetric`, `lineLimit(nil)`, and `ViewThatFits`:
  ```swift
  // ✅ CORRECT: Adaptive layout responding to Dynamic Type
  struct AdaptiveItemRow: View {
      @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24

      var body: some View {
          ViewThatFits(in: .horizontal) {
              // Wide horizontal presentation
              HStack(spacing: 12) {
                  Text("Settings Description")
                      .font(.body)
                  Spacer()
                  Image(systemName: "chevron.right")
                      .frame(width: iconSize, height: iconSize)
              }

              // Compact vertical fallback for large Accessibility text sizes
              VStack(alignment: .leading, spacing: 8) {
                  Text("Settings Description")
                      .font(.body)
                  Image(systemName: "chevron.right")
                      .frame(width: iconSize, height: iconSize)
              }
          }
      }
  }
  ```

---

### Bug 05: Touch Target Collapse & Invisible Tap Dead Zones
- **Root Cause**: SwiftUI views with transparent backgrounds only detect taps directly on rendered text glyphs or image pixels.
- **The Fix**:
  Apply `.contentShape(Rectangle())` and ensure minimum 44×44 pt touch targets:
  ```swift
  // ✅ CORRECT: Fully tappable row with proper hit-testing bounds
  Button(action: handleAction) {
      HStack {
          Image(systemName: "bell.fill")
          Text("Notifications")
          Spacer()
      }
      .frame(minHeight: 44)
      .contentShape(Rectangle())
  }
  .buttonStyle(.plain)
  ```

---

### Bug 06: Flickering & Lost State in `ForEach` Identity
- **Root Cause**: Using array indices `ForEach(0..<items.count, id: \.self)` causes identity collapse when items are reordered or removed.
- **The Fix**:
  Use stable unique identifiers conforming to `Identifiable`:
  ```swift
  // ❌ WRONG: Unstable index identity
  ForEach(0..<items.count, id: \.self) { idx in
      ItemRow(item: items[idx])
  }

  // ✅ CORRECT: Stable model ID preserves scroll position and animations
  ForEach(items) { item in
      ItemRow(item: item)
  }
  ```

---

### Bug 07: ScrollView Frame Stutter with Lazy Stacks
- **Root Cause**: Views inside `LazyVStack` having variable, unestimated heights cause scroll position jumps as items are instantiated.
- **The Fix**:
  Reserve estimated layout bounds or use `scrollTargetLayout()`:
  ```swift
  ScrollView {
      LazyVStack(spacing: 16) {
          ForEach(feedItems) { item in
              FeedCard(item: item)
                  .frame(minHeight: 180) // Provide minimum bounds stability
          }
      }
      .scrollTargetLayout()
  }
  .scrollTargetBehavior(.viewAligned)
  ```

---

### Bug 08: Keyboard Overlap on Form Input Fields
- **Root Cause**: Text fields obscured by the software keyboard on small iPhone screens.
- **The Fix**:
  Enable interactive keyboard dismissal and scroll padding:
  ```swift
  ScrollView {
      VStack(spacing: 20) {
          TextField("Email", text: $email)
          SecureField("Password", text: $password)
      }
      .padding()
  }
  .scrollDismissesKeyboard(.interactively)
  ```

---

### Bug 09: NavigationSplitView Column Collapse Anomaly
- **Root Cause**: On compact iPhone screens, `NavigationSplitView` defaults to showing the detail view before user selection.
- **The Fix**:
  Control column visibility explicitly:
  ```swift
  @State private var columnVisibility = NavigationSplitViewVisibility.all

  NavigationSplitView(columnVisibility: $columnVisibility) {
      SidebarView(selectedItem: $selectedItem)
  } detail: {
      DetailView(item: selectedItem)
  }
  .navigationSplitViewStyle(.balanced)
  ```

---

### Bug 10: Animation Glitch with Conditional View Insertion
- **Root Cause**: Removing views via `if condition { MyView() }` without explicit container bounds causes sibling views to snap abruptly.
- **The Fix**:
  Use asymmetric transitions and `.clipped()` wrappers:
  ```swift
  if isBannerVisible {
      BannerView()
          .transition(.move(edge: .top).combined(with: .opacity))
  }
  ```

---

### Bug 11: Debounced Text Field Two-Way Binding Race
- **Root Cause**: Modifying text input state directly from asynchronous network tasks causes cursor jumps and dropped keystrokes.
- **The Fix**:
  Separate user input state from debounced search query processing via `task(id:)`:
  ```swift
  @State private var rawText: String = ""
  @State private var searchResults: [SearchResult] = []

  TextField("Search...", text: $rawText)
      .task(id: rawText) {
          try? await Task.sleep(nanoseconds: 300_000_000) // 300ms debounce
          guard !Task.isCancelled else { return }
          searchResults = await performSearch(rawText)
      }
  ```

---

### Bug 12: Missing Environment Dependency Runtime Crash
- **Root Cause**: Using `@Environment(Service.self)` without injecting `.environment(service)` in a parent view hierarchy causes immediate crash.
- **The Fix**:
  Inject default instances or verify environment bindings in previews:
  ```swift
  #Preview {
      DashboardView()
          .environment(PreviewData.mockService)
  }
  ```

---

## 5. Pre-Commit SwiftUI Layout Checklist

Before concluding changes to any SwiftUI view:

- [ ] **No Body Mutations**: No `@State` or `@Binding` properties are written within computed `body` properties.
- [ ] **No Hardcoded Screen Sizes**: Zero references to fixed device widths (`width: 375`) or deprecated `UIScreen.main.bounds`.
- [ ] **Dynamic Type Scalable**: Text scales smoothly without clipping when tested with Accessibility Large fonts.
- [ ] **Safe Area Compliant**: Buttons and inputs remain clear of the notch, Island, and Home Indicator.
- [ ] **Interactive Hit Bounds**: All buttons and custom gestures have `.contentShape(Rectangle())` and minimum 44×44 pt targets.
- [ ] **Memory & Render Profiling**: `Self._printChanges()` verifies views do not trigger runaway re-render cycles.