---
name: token-optimizer
description: >-
  Operational protocol for minimizing LLM token consumption and context window bloat during agent coding workflows. Use when: exploring large codebases, inspecting functions, classes, or structs without reading entire files, performing line-range slice viewing (StartLine/EndLine), budgeting context windows for multi-step reasoning, or avoiding context eviction and reasoning degradation on long coding sessions.
---

# Token Optimizer (LLM Context Economics & Surgical Navigation) ⚡️🧠

The definitive operational manual for AI coding agents tasked with navigating, inspecting, modifying, and verifying large codebases while consuming the absolute minimum number of tokens and preserving model reasoning bandwidth.

---

## 1. Executive Summary & Core Philosophy

Large Language Models (LLMs) operate with finite context windows and experience **attention diffusion and reasoning degradation** as context windows fill with hundreds of thousands of irrelevant tokens. When an agent opens an entire 2,000-line file merely to inspect a 15-line function, it exhausts valuable token budgets, increases latency, and significantly degrades its ability to perform sound multi-step reasoning.

1. **Failure Modes of AI Agents**:
   - **Greedy File Reads**: Using `view_file` on entire files without specifying `StartLine` and `EndLine`.
   - **Blind Repository Greps**: Running unconstrained searches across `node_modules/`, `.build/`, or generated lockfiles, flooding the context window with minified javascript or binary artifacts.
   - **Full-File Overwrites**: Regenerating a 1,000-line file with `write_to_file` when only a 3-line surgical patch was needed via `replace_file_content`.
   - **Chat Code Echoing**: Pasting large duplicate blocks of code back to the user when a concise diff or GitHub file link was sufficient.

2. **The Optimizer's Mandate**:
   - **Documentation-First Routing**: Inspect `AGENTS.md` and `ARCHITECTURE.md` to identify the exact 1–2 target files before opening any source files.
   - **Targeted Symbol Navigation**: Pinpoint symbols using line-numbered grep (`git grep -n "func execute"`), then read a tight 30–50 line window slice (`StartLine=X, EndLine=Y`).
   - **AST & Interface-First Inspection**: Inspect protocol and header definitions rather than full method implementations.
   - **Surgical Diff Editing**: Apply contiguous edits via `replace_file_content`.

---

## 2. Mathematical Token Budgeting Framework

```
+-------------------------------------------------------------------------+
|                       CONTEXT WINDOW ALLOCATION RULES                   |
+-------------------------------------------------------------------------+
| Phase 1: Problem Definition & Architecture Routing     | 10% - 15% Max  |
| Phase 2: Surgical Target Symbol Inspection             | 10% - 15% Max  |
| Phase 3: Active Code Reasoning & Synthesis             | 40% - 50% Core |
| Phase 4: Local Verification Execution & Tool Feedback  | 20% - 25%      |
| Reserve Buffer (Prevents Context Truncation)           | 10% Constant   |
+-------------------------------------------------------------------------+
```

---

## 3. The 4 Golden Rules of Surgical Codebase Navigation

```
[Target Investigation Initiated]
               │
               ▼
   1. Check ARCHITECTURE.md or AGENTS.md
         ├── YES ──► Route directly to target directory
         └── NO  ──► Locate target file using shallow find
               │
               ▼
   2. Locate Exact Symbol Line Number:
      git grep -n "class TargetClass" Sources/
               │
               ▼
   3. Slice Read (30-50 lines max):
      view_file StartLine=target-5 EndLine=target+35
               │
               ▼
   4. Apply Contiguous Patch:
      replace_file_content StartLine=X EndLine=Y
               │
               ▼
   5. Verify via Scoped Fast Test
```

### Rule 1: Shallow Directory Surveys
Never run unconstrained recursive `find .` or `ls -R`. Always limit depth and prune heavy directories:
```bash
# ✅ CORRECT: Shallow taxonomy mapping
find . -maxdepth 2 -not -path '*/.*' -not -path './node_modules*' -not -path './.build*'

# ✅ CORRECT: Tree with depth constraint
tree -L 2 -I "node_modules|.build|DerivedData|.git"
```

### Rule 2: Line-Numbered Targeted Grep
Before viewing any file, locate the exact line number of the symbol:
```bash
# ✅ CORRECT: Pinpoint line number with zero context bloat
git grep -n "class AuthenticationService" Sources/
# Output: Sources/Services/AuthenticationService.swift:42: class AuthenticationService
```

### Rule 3: Precise Line Slicing
Once the line number is known, view only the surrounding context:
```bash
# ✅ CORRECT: Consumes ~250 tokens instead of 10,000+ tokens
view_file AbsolutePath="/path/to/AuthenticationService.swift" StartLine=40 EndLine=85
```

### Rule 4: Surgical Contiguous Editing
Avoid full-file rewrites. Replace only the targeted contiguous block:
```json
{
  "StartLine": 55,
  "EndLine": 65,
  "TargetContent": "...",
  "ReplacementContent": "..."
}
```

---

## 4. Context-Saving Shell One-Liners & Output Filters

Unfiltered shell output is one of the primary drivers of context window bloat. Always truncate and filter tool output:

### 4.1 Git Status & Log Filters
```bash
# ❌ BAD: Dumps hundreds of lines
git log
git status

# ✅ CORRECT: Compact one-line summary
git status --short
git log --oneline -n 5
git diff --name-only
```

### 4.2 Test Output Triage
```bash
# ❌ BAD: Dumps 20,000 lines of raw compiler and test output
swift test

# ✅ CORRECT: Filter to test failures only
swift test 2>&1 | grep -E "error:|failed|FAILED|FAILURE" | head -n 25

# ✅ CORRECT: Target specific test class
swift test --filter AuthenticationTests
```

### 4.3 Search Output Filtering
```bash
# ❌ BAD: Thousands of matches across minified bundles
grep -r "token" .

# ✅ CORRECT: Scoped search ignoring build and vendor artifacts
git grep -n -E "auth_token|authToken" -- 'Sources/**/*.swift' ':!*.resolved'
```

---

## 5. Real-World Architectural Case Studies

### Case Study 1: Diagnosing a Monolithic Parser Bug

**Scenario**: A 3,500-line compiler parser (`Sources/Parser/SwiftASTParser.swift`) exhibits a crash during operator precedence parsing. A naive agent would read the entire 3,500 lines (~28,000 tokens), exhausting its context window.

**Surgical Execution**:
1. Locate the operator parsing function via git grep:
   ```bash
   git grep -n "func parseOperatorPrecedence" Sources/Parser/
   # Output: Sources/Parser/SwiftASTParser.swift:1840: func parseOperatorPrecedence() throws -> OperatorNode
   ```
2. Read a 45-line slice:
   ```bash
   view_file AbsolutePath=".../SwiftASTParser.swift" StartLine=1838 EndLine=1883
   ```
3. Identify the missing nil check at line 1862 and replace using `replace_file_content` across lines 1858–1868.
4. Run targeted test:
   ```bash
   swift test --filter OperatorPrecedenceTests
   ```
5. **Total Tokens Consumed**: Under 850 tokens (97% savings compared to full file reading).

---

### Case Study 2: Monorepo Target Discovery

**Scenario**: A monorepo contains 85 packages. A feature requires modifying the caching layer of the auth service.

**Surgical Execution**:
1. Check `ARCHITECTURE.md` or package manifests:
   ```bash
   find packages/ -maxdepth 2 -name "package.json" | grep auth
   # Output: packages/auth-service/package.json
   ```
2. Inspect dependencies inside `packages/auth-service/package.json` (StartLine=1, EndLine=35).
3. Directly edit the targeted module without traversing the remaining 84 packages.

---

## 6. Pre-Flight Token Conservation Checklist

Before dispatching tools or writing code:

- [ ] **Target Confirmed**: Did I identify the specific file and line number before calling `view_file`?
- [ ] **Bounded View Window**: Did I specify `StartLine` and `EndLine` (≤ 50 lines) in `view_file`?
- [ ] **Filtered Shell Output**: Are long-running or verbose commands piped to `head`, `tail`, or `grep`?
- [ ] **Surgical Edits**: Am I using `replace_file_content` instead of rewriting entire files?
- [ ] **Concise Communication**: Are my explanations concise and referencing file links rather than repeating large blocks of source code?