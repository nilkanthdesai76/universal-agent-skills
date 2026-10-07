---
name: git-atomic-curator
description: >-
  Operational protocol for structuring, staging, and committing git changes. Use when: staging multiple modified files across a repository, organizing messy uncommitted working trees into clean atomic commits, adhering to Conventional Commits (chore, feat, fix, refactor, test, docs, ci), ensuring every intermediate commit compiles for git bisect safety, or structuring pull request commit histories.
---

# Git Atomic Curator (Commit Hygiene & History Engineering) 🌳⚡️

The definitive operational manual for AI coding agents tasked with structuring, staging, committing, and maintaining immaculate, bisect-safe git histories.

---

## 1. Executive Summary & Core Philosophy

A git repository's commit log is not merely a backup log; it is an executable, bisectable historical record of engineering decisions. Naive agents routinely fail git hygiene by dumping dozens of unrelated modified files into a single monolithic commit with vague messages like `"updates"` or `"fixed bugs"`.

1. **Failure Modes of AI Agents**:
   - **Monolithic Dumps**: Committing features, bug fixes, formatting changes, and documentation in a single commit.
   - **Bisect-Breaking Commits**: Committing a change in step 1 that does not compile until step 3, making `git bisect` impossible for debugging regressions.
   - **Leaked Working Artifacts**: Accidental staging of `.DS_Store`, build outputs, or `.env` credential files.
   - **Vague Commit Messages**: Non-conventional, non-searchable commit headers lacking scope and context.

2. **The Curator's Mandate**:
   - **Atomic Invariant**: Every single commit represents ONE logical change.
   - **Compilability Invariant**: Every single commit MUST compile cleanly and pass existing tests.
   - **Conventional Commits Standard**: Format: `<type>(<scope>): <concise description>`.
   - **Bisect Resilience**: Any past commit checked out via `git checkout <sha>` must build without compilation errors.

---

## 2. Conventional Commits 1.0.0 Specification

Format:
```
<type>(<optional scope>): <description>

[optional body explaining motivation and contrasting with previous behavior]

[optional footer(s) such as BREAKING CHANGE: ... or Closes #123]
```

### 2.1 Commit Types & Use Cases

| Type | Intent | Example |
|---|---|---|
| `feat` | Introduces a new user-facing or public API capability | `feat(auth): add biometric FaceID authentication provider` |
| `fix` | Patches a bug or regression | `fix(layout): resolve safe area overlap on iPad split view` |
| `refactor` | Code restructuring without altering external behavior | `refactor(network): extract retry logic into reusable middleware` |
| `test` | Adding or updating unit, integration, or snapshot tests | `test(player): add concurrency race tests for queue manager` |
| `docs` | Documentation, comments, or architecture diagrams only | `docs(readme): add simulator execution flags and badge link` |
| `ci` | Modifying CI/CD pipelines, workflows, or runner scripts | `ci(actions): add macOS 15 runner matrix and SPM cache step` |
| `chore` | Dependency bumps, repo housekeeping, tooling updates | `chore(deps): update SwiftLint configuration to 0.55` |
| `perf` | Performance optimization without changing behavior | `perf(cache): replace sync disk writes with background actor queue` |

---

## 3. Systematic Staging & Atomic Decomposition Flow

When an agent finds multiple modified, deleted, or untracked files in a working tree, follow this staged flow:

```
[ Messy Working Tree: 15 Modified Files ]
                   │
                   ▼
  1. Inspect Status: git status -s
                   │
                   ▼
  2. Group by Domain & Dependency Order:
     ├── Layer 1: Core Models & Schemas  ──► Commit: feat(models): ...
     ├── Layer 2: Business Logic/Actor   ──► Commit: feat(core): ...
     ├── Layer 3: Presentation / Views   ──► Commit: feat(ui): ...
     ├── Layer 4: Unit / Mock Tests      ──► Commit: test(core): ...
     └── Layer 5: Docs & Workflows       ──► Commit: docs(api): ...
                   │
                   ▼
  3. Verify Each Intermediate Commit:
     Run build/test command after EVERY commit.
```

---

## 4. Real-World Staging Recipes & Workflows

### 4.1 Recipe 1: Decomposing a Full-Stack Feature

Suppose your working tree contains:
- `Sources/Models/UserProfile.swift` (new model)
- `Sources/Services/AuthService.swift` (actor service)
- `Sources/UI/LoginView.swift` (SwiftUI view)
- `Tests/AuthServiceTests.swift` (tests)
- `README.md` (updated docs)

**Execution:**

```bash
# Step 1: Commit model layer
git add Sources/Models/UserProfile.swift
git commit -m "feat(models): define UserProfile model and validation rules"
swift build

# Step 2: Commit service layer
git add Sources/Services/AuthService.swift
git commit -m "feat(auth): implement AuthService actor with token refresh"
swift build

# Step 3: Commit UI layer
git add Sources/UI/LoginView.swift
git commit -m "feat(ui): implement LoginView with async authentication flow"
swift build

# Step 4: Commit test suite
git add Tests/AuthServiceTests.swift
git commit -m "test(auth): add unit tests for AuthService token expiry"
swift test

# Step 5: Commit documentation
git add README.md
git commit -m "docs(auth): document authentication setup and environment keys"
```

---

### 4.2 Recipe 2: Interactive Patch Staging (`git add -p`)

When a single file contains both a bug fix and unrelated formatting or refactoring, stage them separately:

```bash
# Interactively stage specific hunks
git add -p Sources/Core/NetworkManager.swift

# Hunk commands:
# 'y' - stage this hunk
# 'n' - do not stage this hunk
# 's' - split the hunk into smaller hunks
# 'e' - manually edit the hunk
```

Commit the fix first:
```bash
git commit -m "fix(network): handle HTTP 429 rate limit backoff correctly"
```

Then stage the remaining changes:
```bash
git add Sources/Core/NetworkManager.swift
git commit -m "refactor(network): modernize URLSession delegate configuration"
```

---

### 4.3 Recipe 3: Automated Bisect Verification

To ensure intermediate commits never break historical regression hunts, run `git bisect` automated testing:

```bash
# Start bisect session
git bisect start HEAD v1.0.0

# Run automated test script against each checked-out commit
git bisect run swift test --filter RegressionTests

# Reset back to original HEAD after identifying offending commit
git bisect reset
```

---

## 5. History Editing & Hygiene Maintenance

### 5.1 Amending Mistakes Without History Pollution
If you made a typo in the last commit message or forgot a file before pushing:

```bash
# Add forgotten file to staging
git add Sources/Core/MissingFile.swift

# Amend into previous commit without changing message
git commit --amend --no-edit

# Or amend commit message
git commit --amend -m "fix(core): correct parameter validation in AuthService"
```

> **Warning**: Never amend commits that have already been pushed to a shared remote tracking branch (`origin/main`), unless working on an isolated feature branch.

### 5.2 Rewording or Reordering Past Commits (`git rebase -i`)

To tidy up the last 4 commits on a feature branch:

```bash
git rebase -i HEAD~4
```

Editor options:
- `pick`: Use commit as-is
- `reword`: Change commit message
- `edit`: Stop to modify files in this commit
- `squash`: Meld into previous commit
- `fixup`: Meld into previous commit, discarding this commit's message
- `drop`: Delete commit entirely

---

## 6. Pre-Commit Verification Checklist

Before pushing a branch or submitting a pull request, run this verification:

- [ ] `git status` returns completely clean (nothing unstaged, nothing untracked).
- [ ] `git log --oneline -5` displays clear Conventional Commit prefixes (`feat:`, `fix:`, `refactor:`, `test:`).
- [ ] No temporary files (`.DS_Store`, `*.tmp`, `DerivedData`) are present in `git log --stat -1`.
- [ ] Each intermediate commit builds independently when checked out.
- [ ] Sensitive files (`.env`, private keys) were not committed to git history.