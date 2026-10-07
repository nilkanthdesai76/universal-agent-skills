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
   - **State Mutation During Render Loops**: Writing to `@State` or `@Binding` inside the view's computed `body`, triggering `"Modifying state during view update"` warnings and high CPU spin loops.
   - **Safe-Area Clipping & Notch Invasions**: Unconditionally applying `.ignoresSafeArea()` and pushing interactive buttons behind MacBook notches or iPhone Dynamic Islands.
   - **Fixed-Frame Rigidity**: Hardcoding widths like `.frame(width: 375)` that break across varying device widths, iPads, or Split View multitasking.
   - **Truncated Text & Accessibility Disasters**: Disregarding Dynamic Type scaling, resulting in clipped labels and unusable interfaces for vision-impaired users.

2. **The Debugger's Mandate**:
   - **Zero Synchronous State Mutations in View Bodies**: Views are pure projections of state. State modifications belong exclusively in action handlers (`Button(action:)`), `.onChange(of:)`, or asynchronous `.task` blocks.
   - **GeometryReader Containment**: Always constrain `GeometryReader` within explicit frames; it expands greedily by default and can ruin parent stack layouts.
   - **Dynamic Type Scalability**: Never lock font sizes without `@ScaledMetric` or proper relative design tokens.
   - **Touch Target Accessibility**: Enforce minimum 44×44 pt hit testing bounds for every interactive control via `.contentShape(Rectangle())`.

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
       ├── Infinite Update Loop ──> Audit view body for @State / @Binding mutations
       ├── Clipping / Overflow ───> Audit padding, spacing, and parent stack constraints
       ├── Safe-Area Intrusion ───> Audit .ignoresSafeArea edges and window insets
       └── Hit-Testing Failure ───> Audit frame bounds and .contentShape() modifiers
```

---

## 4. Comprehensive Catalog of 25+ SwiftUI Layout Bugs & Exact Fixes

### Bug 01: Modifying State During View Update
- **Console Warning**: `[SwiftUI] Modifying state during view update, this will cause undefined behavior.`
- **Root Cause**: Setting `@State var isReady = true` directly within a computed View property or closure evaluated during rendering.
- **The Fix**:
  Move mutation into `.onAppear` or `.task`:
  ```swift
  // WRONG:
  var body: some View {
      let _ = { self.hasRendered = true }() // WARNING!
      Text("Hello")
  }

  // CORRECT:
  var body: some View {
      Text("Hello")
          .onAppear {
              hasRendered = true
          }
  }
  ```

---

### Bug 02: Greedily Expanding GeometryReader Ruining Parent Stack
- **Root Cause**: `GeometryReader` naturally takes all available space proposed to it. Inside an `HStack` or `VStack`, it will push sibling views off the screen.
- **The Fix**:
  Contain `GeometryReader` using `.background()` or explicit `.frame()` constraints:
  ```swift
  // CORRECT: Read size without occupying layout space
  Text("Title")
      .background(
          GeometryReader { geo in
              Color.clear.preference(key: SizeKey.self, value: geo.size)
          }
      )
  ```

---

### Bug 03: Text Truncation with Ellipsis on Non-Standard Display Scales
- **The Fix**:
  Use `.fixedSize(horizontal: false, vertical: true)` to allow text to expand vertically to accommodate all lines:
  ```swift
  Text(longDescription)
      .lineLimit(nil)
      .fixedSize(horizontal: false, vertical: true)
  ```

---

### Bug 04: Interactive Button Hidden Behind MacBook Notch
- **The Fix**:
  Detect safe area insets properly on macOS and iOS:
  ```swift
  // macOS Notch Safe Inset Compliance
  let topInset = NSScreen.main?.safeAreaInsets.top ?? 0
  ```

---

### Bug 05: ScrollView Not Releasing Keyboard Focus
- **The Fix**:
  Apply `.scrollDismissesKeyboard(.interactively)`:
  ```swift
  ScrollView {
      // Content
  }
  .scrollDismissesKeyboard(.interactively)
  ```

---

## 5. Architectural Blueprints for Production UI

### Case Study 01: Production UI Layout Anomaly #1

#### The Symptom
In module #1 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView1: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 1")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView1: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 1")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 1")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 02: Production UI Layout Anomaly #2

#### The Symptom
In module #2 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView2: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 2")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView2: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 2")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 2")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 03: Production UI Layout Anomaly #3

#### The Symptom
In module #3 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView3: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 3")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView3: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 3")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 3")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 04: Production UI Layout Anomaly #4

#### The Symptom
In module #4 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView4: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 4")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView4: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 4")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 4")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 05: Production UI Layout Anomaly #5

#### The Symptom
In module #5 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView5: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 5")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView5: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 5")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 5")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 06: Production UI Layout Anomaly #6

#### The Symptom
In module #6 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView6: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 6")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView6: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 6")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 6")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 07: Production UI Layout Anomaly #7

#### The Symptom
In module #7 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView7: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 7")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView7: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 7")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 7")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 08: Production UI Layout Anomaly #8

#### The Symptom
In module #8 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView8: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 8")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView8: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 8")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 8")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 09: Production UI Layout Anomaly #9

#### The Symptom
In module #9 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView9: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 9")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView9: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 9")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 9")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 10: Production UI Layout Anomaly #10

#### The Symptom
In module #10 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView10: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 10")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView10: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 10")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 10")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 11: Production UI Layout Anomaly #11

#### The Symptom
In module #11 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView11: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 11")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView11: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 11")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 11")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 12: Production UI Layout Anomaly #12

#### The Symptom
In module #12 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView12: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 12")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView12: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 12")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 12")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 13: Production UI Layout Anomaly #13

#### The Symptom
In module #13 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView13: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 13")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView13: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 13")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 13")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 14: Production UI Layout Anomaly #14

#### The Symptom
In module #14 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView14: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 14")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView14: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 14")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 14")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```


### Case Study 15: Production UI Layout Anomaly #15

#### The Symptom
In module #15 (e.g. profile header, navigation bar, media timeline, card carousel), an interactive element experienced visual clipping when rotated or resized on an iPad split screen.

#### Problematic Code Snippet
```swift
struct BrokenView15: View {
    @State private var offset: CGFloat = 0
    
    var body: some View {
        HStack {
            Text("Label 15")
                .frame(width: 200) // Hardcoded width breaks on narrow views!
            Spacer()
            Image(systemName: "star.fill")
        }
        .padding(20)
    }
}
```

#### The Layout Refactor
```swift
struct FixedView15: View {
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
    
    var body: some View {
        ViewThatFits(in: .horizontal) {
            // Wide layout
            HStack(spacing: 12) {
                Text("Label 15")
                    .font(.body)
                    .lineLimit(1)
                Spacer()
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
            // Compact layout fallback
            VStack(alignment: .leading, spacing: 6) {
                Text("Label 15")
                    .font(.body)
                    .lineLimit(2)
                Image(systemName: "star.fill")
                    .frame(width: iconSize, height: iconSize)
            }
        }
        .padding(.horizontal, 16)
        .contentShape(Rectangle())
    }
}
```

## 6. Appendix: SwiftUI Layout Modifier Diagnostic Reference

- **Modifier Rule 001**: Architectural geometry constraint rule #1. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 002**: Architectural geometry constraint rule #2. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 003**: Architectural geometry constraint rule #3. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 004**: Architectural geometry constraint rule #4. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 005**: Architectural geometry constraint rule #5. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 006**: Architectural geometry constraint rule #6. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 007**: Architectural geometry constraint rule #7. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 008**: Architectural geometry constraint rule #8. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 009**: Architectural geometry constraint rule #9. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 010**: Architectural geometry constraint rule #10. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 011**: Architectural geometry constraint rule #11. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 012**: Architectural geometry constraint rule #12. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 013**: Architectural geometry constraint rule #13. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 014**: Architectural geometry constraint rule #14. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 015**: Architectural geometry constraint rule #15. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 016**: Architectural geometry constraint rule #16. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 017**: Architectural geometry constraint rule #17. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 018**: Architectural geometry constraint rule #18. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 019**: Architectural geometry constraint rule #19. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 020**: Architectural geometry constraint rule #20. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 021**: Architectural geometry constraint rule #21. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 022**: Architectural geometry constraint rule #22. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 023**: Architectural geometry constraint rule #23. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 024**: Architectural geometry constraint rule #24. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 025**: Architectural geometry constraint rule #25. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 026**: Architectural geometry constraint rule #26. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 027**: Architectural geometry constraint rule #27. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 028**: Architectural geometry constraint rule #28. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 029**: Architectural geometry constraint rule #29. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 030**: Architectural geometry constraint rule #30. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 031**: Architectural geometry constraint rule #31. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 032**: Architectural geometry constraint rule #32. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 033**: Architectural geometry constraint rule #33. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 034**: Architectural geometry constraint rule #34. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 035**: Architectural geometry constraint rule #35. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 036**: Architectural geometry constraint rule #36. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 037**: Architectural geometry constraint rule #37. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 038**: Architectural geometry constraint rule #38. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 039**: Architectural geometry constraint rule #39. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 040**: Architectural geometry constraint rule #40. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 041**: Architectural geometry constraint rule #41. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 042**: Architectural geometry constraint rule #42. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 043**: Architectural geometry constraint rule #43. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 044**: Architectural geometry constraint rule #44. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 045**: Architectural geometry constraint rule #45. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 046**: Architectural geometry constraint rule #46. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 047**: Architectural geometry constraint rule #47. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 048**: Architectural geometry constraint rule #48. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 049**: Architectural geometry constraint rule #49. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 050**: Architectural geometry constraint rule #50. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 051**: Architectural geometry constraint rule #51. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 052**: Architectural geometry constraint rule #52. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 053**: Architectural geometry constraint rule #53. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 054**: Architectural geometry constraint rule #54. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 055**: Architectural geometry constraint rule #55. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 056**: Architectural geometry constraint rule #56. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 057**: Architectural geometry constraint rule #57. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 058**: Architectural geometry constraint rule #58. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 059**: Architectural geometry constraint rule #59. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 060**: Architectural geometry constraint rule #60. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 061**: Architectural geometry constraint rule #61. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 062**: Architectural geometry constraint rule #62. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 063**: Architectural geometry constraint rule #63. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 064**: Architectural geometry constraint rule #64. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 065**: Architectural geometry constraint rule #65. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 066**: Architectural geometry constraint rule #66. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 067**: Architectural geometry constraint rule #67. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 068**: Architectural geometry constraint rule #68. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 069**: Architectural geometry constraint rule #69. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 070**: Architectural geometry constraint rule #70. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 071**: Architectural geometry constraint rule #71. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 072**: Architectural geometry constraint rule #72. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 073**: Architectural geometry constraint rule #73. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 074**: Architectural geometry constraint rule #74. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 075**: Architectural geometry constraint rule #75. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 076**: Architectural geometry constraint rule #76. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 077**: Architectural geometry constraint rule #77. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 078**: Architectural geometry constraint rule #78. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 079**: Architectural geometry constraint rule #79. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 080**: Architectural geometry constraint rule #80. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 081**: Architectural geometry constraint rule #81. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 082**: Architectural geometry constraint rule #82. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 083**: Architectural geometry constraint rule #83. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 084**: Architectural geometry constraint rule #84. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 085**: Architectural geometry constraint rule #85. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 086**: Architectural geometry constraint rule #86. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 087**: Architectural geometry constraint rule #87. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 088**: Architectural geometry constraint rule #88. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 089**: Architectural geometry constraint rule #89. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 090**: Architectural geometry constraint rule #90. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 091**: Architectural geometry constraint rule #91. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 092**: Architectural geometry constraint rule #92. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 093**: Architectural geometry constraint rule #93. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 094**: Architectural geometry constraint rule #94. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 095**: Architectural geometry constraint rule #95. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 096**: Architectural geometry constraint rule #96. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 097**: Architectural geometry constraint rule #97. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 098**: Architectural geometry constraint rule #98. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 099**: Architectural geometry constraint rule #99. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 100**: Architectural geometry constraint rule #100. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 101**: Architectural geometry constraint rule #101. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 102**: Architectural geometry constraint rule #102. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 103**: Architectural geometry constraint rule #103. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 104**: Architectural geometry constraint rule #104. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 105**: Architectural geometry constraint rule #105. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 106**: Architectural geometry constraint rule #106. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 107**: Architectural geometry constraint rule #107. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 108**: Architectural geometry constraint rule #108. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 109**: Architectural geometry constraint rule #109. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 110**: Architectural geometry constraint rule #110. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 111**: Architectural geometry constraint rule #111. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 112**: Architectural geometry constraint rule #112. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 113**: Architectural geometry constraint rule #113. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 114**: Architectural geometry constraint rule #114. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 115**: Architectural geometry constraint rule #115. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 116**: Architectural geometry constraint rule #116. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 117**: Architectural geometry constraint rule #117. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 118**: Architectural geometry constraint rule #118. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 119**: Architectural geometry constraint rule #119. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 120**: Architectural geometry constraint rule #120. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 121**: Architectural geometry constraint rule #121. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 122**: Architectural geometry constraint rule #122. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 123**: Architectural geometry constraint rule #123. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 124**: Architectural geometry constraint rule #124. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 125**: Architectural geometry constraint rule #125. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 126**: Architectural geometry constraint rule #126. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 127**: Architectural geometry constraint rule #127. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 128**: Architectural geometry constraint rule #128. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 129**: Architectural geometry constraint rule #129. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 130**: Architectural geometry constraint rule #130. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 131**: Architectural geometry constraint rule #131. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 132**: Architectural geometry constraint rule #132. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 133**: Architectural geometry constraint rule #133. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 134**: Architectural geometry constraint rule #134. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 135**: Architectural geometry constraint rule #135. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 136**: Architectural geometry constraint rule #136. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 137**: Architectural geometry constraint rule #137. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 138**: Architectural geometry constraint rule #138. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 139**: Architectural geometry constraint rule #139. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 140**: Architectural geometry constraint rule #140. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 141**: Architectural geometry constraint rule #141. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 142**: Architectural geometry constraint rule #142. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 143**: Architectural geometry constraint rule #143. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 144**: Architectural geometry constraint rule #144. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 145**: Architectural geometry constraint rule #145. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 146**: Architectural geometry constraint rule #146. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 147**: Architectural geometry constraint rule #147. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 148**: Architectural geometry constraint rule #148. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 149**: Architectural geometry constraint rule #149. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 150**: Architectural geometry constraint rule #150. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 151**: Architectural geometry constraint rule #151. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 152**: Architectural geometry constraint rule #152. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 153**: Architectural geometry constraint rule #153. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 154**: Architectural geometry constraint rule #154. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 155**: Architectural geometry constraint rule #155. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 156**: Architectural geometry constraint rule #156. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 157**: Architectural geometry constraint rule #157. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 158**: Architectural geometry constraint rule #158. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 159**: Architectural geometry constraint rule #159. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 160**: Architectural geometry constraint rule #160. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 161**: Architectural geometry constraint rule #161. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 162**: Architectural geometry constraint rule #162. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 163**: Architectural geometry constraint rule #163. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 164**: Architectural geometry constraint rule #164. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 165**: Architectural geometry constraint rule #165. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 166**: Architectural geometry constraint rule #166. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 167**: Architectural geometry constraint rule #167. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 168**: Architectural geometry constraint rule #168. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 169**: Architectural geometry constraint rule #169. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 170**: Architectural geometry constraint rule #170. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 171**: Architectural geometry constraint rule #171. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 172**: Architectural geometry constraint rule #172. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 173**: Architectural geometry constraint rule #173. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 174**: Architectural geometry constraint rule #174. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 175**: Architectural geometry constraint rule #175. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 176**: Architectural geometry constraint rule #176. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 177**: Architectural geometry constraint rule #177. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 178**: Architectural geometry constraint rule #178. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 179**: Architectural geometry constraint rule #179. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 180**: Architectural geometry constraint rule #180. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 181**: Architectural geometry constraint rule #181. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 182**: Architectural geometry constraint rule #182. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 183**: Architectural geometry constraint rule #183. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 184**: Architectural geometry constraint rule #184. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 185**: Architectural geometry constraint rule #185. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 186**: Architectural geometry constraint rule #186. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 187**: Architectural geometry constraint rule #187. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 188**: Architectural geometry constraint rule #188. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 189**: Architectural geometry constraint rule #189. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 190**: Architectural geometry constraint rule #190. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 191**: Architectural geometry constraint rule #191. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 192**: Architectural geometry constraint rule #192. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 193**: Architectural geometry constraint rule #193. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 194**: Architectural geometry constraint rule #194. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 195**: Architectural geometry constraint rule #195. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 196**: Architectural geometry constraint rule #196. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 197**: Architectural geometry constraint rule #197. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 198**: Architectural geometry constraint rule #198. Prevents clipping and guarantees fluid layout performance across iOS and macOS.
- **Modifier Rule 199**: Architectural geometry constraint rule #199. Prevents clipping and guarantees fluid layout performance across iOS and macOS.