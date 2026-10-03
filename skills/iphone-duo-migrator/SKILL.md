---
name: iphone-duo-migrator
description: >-
  Audits iOS and iPadOS applications for iPhone Duo and foldable dual-screen readiness. Replaces frozen UIScreen.main calls, verifies crease hinge avoidance with ReservedRegion, and adapts dual-pane layouts with ArrangementView.
---

# iPhone Duo Migrator Skill

This skill provides step-by-step guidance for updating iOS apps to comply with Apple's iPhone Duo and foldable layout guidelines.

## Migration Checklist

1. **Step 1: Eliminate `UIScreen.main` Reads**:
   - Search the codebase for `UIScreen.main.bounds` or `UIScreen.main.bounds.width`.
   - Replace with `UIWindowScene.coordinateSpace.bounds` or SwiftUI `GeometryReader` / `WindowSceneBounds`.

2. **Step 2: Avoid the Hinge Seam (`ReservedRegion`)**:
   - Ensure floating buttons, persistent toolbars, or video control bars do not land in the 14pt center crease.
   - Wrap floating controls in `.avoidFoldHinge()`.

3. **Step 3: Dual-Pane Refactoring (`ArrangementView`)**:
   - For List-Detail or Inspector interfaces, adopt `ArrangementView(primary: { ... }, secondary: { ... })`.
   - Renders side-by-side on unfolded/book posture and collapses to a single screen on the outer display.

4. **Step 4: Simulator Verification**:
   - Run in Xcode Duo Simulator under Device Hub.
   - Test across `.flat`, `.book`, `.tabletop`, and `.folded` postures.
