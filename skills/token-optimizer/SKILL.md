---
name: token-optimizer
description: >-
  Minimizes LLM token consumption and context bloat by enforcing codebase-first documentation lookups, targeted symbol searches, and surgical file slice viewing. Use on every coding and research task.
---

# Token Optimizer Skill

This skill teaches AI agents how to maximize reasoning quality while slashing token consumption by up to 80%.

## Core Principles

1. **Manifest-First Traversal**:
   - Before listing whole directories or grepping across hundreds of files, check if `AGENTS.md` or `ARCHITECTURE.md` exists in the repository root.
   - Read the directory taxonomy in `ARCHITECTURE.md` to pinpoint the exact 1–2 files that contain the relevant logic.

2. **Surgical Slice Reading**:
   - Never load entire 500+ line files into context when checking a specific function or struct.
   - Use targeted range reads (`StartLine` and `EndLine`) or grep for specific symbol signatures.

3. **Incremental Patching**:
   - Make precise diff edits rather than regenerating whole source files.
   - Preserves token budget for multi-step reasoning and deep verification.

4. **Verify Locally, Don't Guess**:
   - Run the project's fast test command (specified in `AGENTS.md`) rather than generating dozens of exploratory speculative code variations.
