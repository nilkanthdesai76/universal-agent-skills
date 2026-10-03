# DESIGN_SYSTEM.md — UI Tokens & Design Guidelines

> This document defines visual tokens, component usage, and layout hierarchy.

---

## 🎨 Color Palette & Theming

- **Dark Mode First**: Support native Light and Dark appearances using dynamic semantic colors (e.g. `Color(.systemBackground)` / CSS variables).
- **Vibrancy & Materials**: Use system ultra-thin / frosted glass materials for overlays and floating panels.

---

## 📐 Typography & Spacing Scale

- **Base Spacing Grid**: 4pt / 8pt grid (`4`, `8`, `12`, `16`, `24`, `32`, `48`).
- **Dynamic Type**: All text elements MUST scale gracefully with system accessibility text sizes.
- **Touch Targets**: Minimum interactive touch target size is `44x44 pt` (Apple HIG) / `48x48 dp` (Material).
