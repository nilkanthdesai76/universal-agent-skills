---
name: codebase-bootstrap
description: >-
  Audits an existing codebase and automatically bootstraps the 7 essential documentation files (AGENTS.md, ARCHITECTURE.md, PRD.md, TESTING.md, CODE_STYLE.md, SECURITY.md, DESIGN_SYSTEM.md) to supercharge future AI agent interactions and reduce token waste.
---

# Codebase Bootstrap Skill

When entering a new or undocumented project, use this skill to quickly generate the foundational documentation hierarchy.

## Execution Workflow

1. **Inspect Repo Manifest**:
   - Check `Package.swift`, `package.json`, `Cargo.toml`, or `requirements.txt` to identify the tech stack, entry point, and test runners.

2. **Generate `AGENTS.md`**:
   - Populate with build, test, and lint commands.
   - List non-negotiable architectural rules.

3. **Generate `ARCHITECTURE.md`**:
   - Map top-level directories to their architectural roles.
   - Draw an ASCII or Mermaid diagram of data flow.

4. **Generate `TESTING.md` & `CODE_STYLE.md`**:
   - Document how to run unit tests and linters.

5. **Commit the Documentation**:
   - Commit these files as `docs: bootstrap agent-first documentation suite`.
