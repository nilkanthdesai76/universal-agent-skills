# AGENTS.md — Agent Ground Rules & Operational Manual

> This file is the primary entry point for AI coding assistants (Antigravity, Cursor, Claude Code, Windsurf, Copilot, Kimi).
> Before inspecting source code or calling tools, agents MUST consult this document.

---

## 🧭 Repository Overview

- **Project Name**: `<PROJECT_NAME>`
- **Core Purpose**: `<1-2 sentence description of what the project solves>`
- **Primary Tech Stack**: `<e.g. Swift 6, SwiftUI, AppKit / TypeScript, React, Next.js / Python, FastAPI>`
- **Deployment Targets**: `<e.g. iOS 16+, macOS 13+ / Node 20+, Docker>`

---

## ⚡ Essential Commands (Copy-Paste Ready)

```bash
# Build project
<BUILD_COMMAND>

# Run full test suite
<TEST_COMMAND>

# Run single targeted test
<SINGLE_TEST_COMMAND>

# Format & Lint
<LINT_COMMAND>
```

---

## 🛡️ Critical Invariants & Non-Negotiable Rules

1. **Token Efficiency First**:
   - Do NOT search or grep blindly across the whole repository. Consult [ARCHITECTURE.md](ARCHITECTURE.md) to locate the exact module first.
   - Use line-range slice viewing rather than reading entire 1,000-line files.
2. **Preserve Documentation & Comments**:
   - Maintain all existing comments, copyright notices, and docstrings unless explicitly directed to modify them.
3. **Strict Secret Hygiene**:
   - NEVER commit API keys, private tokens, passwords, or internal URLs. Always use environment variables or mock configs.
4. **Verification Gate**:
   - Always run the relevant test suite or build command after making edits to guarantee zero regressions before completing turns.
