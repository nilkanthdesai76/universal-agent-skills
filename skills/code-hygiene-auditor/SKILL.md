---
name: code-hygiene-auditor
description: >-
  Operational protocol for conducting comprehensive pre-commit quality, security, and repository hygiene audits. Use when: performing pre-commit verification, verifying zero compiler warnings (-warnings-as-errors), ensuring no untracked scratch files, test binaries, or .DS_Store files linger in the working tree, auditing .gitignore coverage, and verifying clean test suite passes before concluding tasks.
---

# Code Hygiene Auditor (Pre-Commit Quality & Security Verification) 🧹🔍

The definitive operational manual for AI coding agents tasked with performing strict pre-flight quality, security, and cleanliness audits prior to committing code or concluding tasks.

---

## 1. Executive Summary & Core Philosophy

Clean code hygiene is the firewall that prevents technical debt, leaked credentials, broken builds, and messy git histories from entering production repositories.

1. **Failure Modes of AI Agents**:
   - Leaving unused debug print statements (`print()`, `console.log()`), scratch files, or test binaries in working directories.
   - Overlooking compiler warnings that signify impending data races or deprecated API usage.
   - Forgetting to run `git status` before finishing a task, accidentally committing `.DS_Store` or `.env` files.
   - Disabling linters or silencing type checkers with `@ts-ignore` or `as any` instead of fixing the root cause.

2. **The Auditor's Mandate**:
   - **Zero Secrets**: Exhaustive regex scanning across all modified files before every commit.
   - **Zero Compiler Warnings**: Builds must compile cleanly with zero warnings under modern strict compiler flags (`-warnings-as-errors`).
   - **Immaculate Working Tree**: No untracked temporary files, scratch artifacts, or orphaned build folders left behind.
   - **Green Test Suite**: All unit and regression tests must pass deterministically before concluding work.

---

## 2. The 5-Pillar Code Hygiene Framework

```
+-----------------------------------------------------------------------------------------+
|                                5-PILLAR HYGIENE FRAMEWORK                               |
+---------------------+-------------------------------------------------------------------+
| Pillar              | Verification Objective & Standard                                 |
+---------------------+-------------------------------------------------------------------+
| 1. Security         | 0 credentials, 0 private keys, 0 unencrypted secrets in git diff  |
| 2. Compilation      | 0 errors, 0 compiler warnings, strict type and concurrency checks |
| 3. Test Suite       | 100% test pass rate with deterministic assertion coverage         |
| 4. Working Tree     | Clean git status; all build artifacts properly gitignored         |
| 5. Git Commit       | Atomic staging, Conventional Commits prefix, descriptive scope    |
+---------------------+-------------------------------------------------------------------+
```

---

## 3. Language-Specific Compiler Warning Enforcement

Every production codebase managed by an AI agent must configure warnings-as-errors to prevent technical rot:

### 3.1 Swift 6 & Apple Platforms
Enforce zero warnings and strict concurrency in `Package.swift`:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "CoreEngine",
    platforms: [.iOS(.v17), .macOS(.v14)],
    products: [
        .library(name: "CoreEngine", targets: ["CoreEngine"]),
    ],
    targets: [
        .target(
            name: "CoreEngine",
            swiftSettings: [
                .unsafeFlags(["-warnings-as-errors"]),
                .enableUpcomingFeature("StrictConcurrency"),
                .enableUpcomingFeature("ExistentialAny"),
                .enableUpcomingFeature("InternalImportsByDefault"),
            ]
        ),
        .testTarget(
            name: "CoreEngineTests",
            dependencies: ["CoreEngine"],
            swiftSettings: [
                .unsafeFlags(["-warnings-as-errors"])
            ]
        ),
    ]
)
```

CLI Build Command:
```bash
swift build -Xswiftc -warnings-as-errors
```

### 3.2 TypeScript / Node.js
Strict type checking configuration in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true
  }
}
```

CLI Typecheck Command:
```bash
pnpm tsc --noEmit
```

### 3.3 Rust Systems
Configure Cargo to treat all compiler and clippy warnings as compilation errors:

```bash
# In CI and pre-commit checks:
RUSTFLAGS="-D warnings" cargo check --all-targets --all-features
cargo clippy --all-targets --all-features -- -D warnings
```

### 3.4 Python Backend
Strict linting and type checking via `ruff` and `mypy`:

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
    "SIM", # flake8-simplify
]
deny-warnings = true

[tool.mypy]
strict = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
```

CLI Audit Command:
```bash
uv run ruff check .
uv run mypy .
```

### 3.5 Go Microservices
Configure `golangci-lint` with strict defaults:

```bash
golangci-lint run --enable-all -D gochecknoglobals,godox
```

---

## 4. Production Pre-Commit Hygiene Audit Script

Include this automated pre-commit audit runner in `scripts/audit-hygiene.sh` to execute before every commit:

```bash
#!/usr/bin/env bash
# audit-hygiene.sh — Pre-flight quality and security validator
set -euo pipefail

echo "🔍 Starting Pre-Flight Code Hygiene Audit..."
FAILURES=0

# 1. Check for uncommitted merge conflict markers
echo -n "  Checking for merge conflict markers... "
if git grep -E "^(<<<<<<<|=======|>>>>>>>)" -- ':!*.md' ':!*.resolved' 2>/dev/null; then
  echo "❌ FAILED: Merge conflict markers found in tracked files!"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Clean"
fi

# 2. Check for leftover debug statements
echo -n "  Checking for leftover debug statements... "
DEBUG_MATCHES=$(git diff --cached -S "console.log" -S "print(" -S "dbg!" -S "fmt.Println" -- ':!*test*' ':!*Test*' 2>/dev/null || true)
if [ -n "$DEBUG_MATCHES" ]; then
  echo "⚠️ WARNING: Possible debug print statements staged in diff:"
  echo "$DEBUG_MATCHES" | head -n 5
else
  echo "✅ Clean"
fi

# 3. Check for lingering scratch files
echo -n "  Checking for untracked scratch/temporary files... "
UNTRACKED=$(git status --porcelain | grep -E "^\?\?" | grep -E "\.(tmp|scratch|log|DS_Store|swp)$" || true)
if [ -n "$UNTRACKED" ]; then
  echo "❌ FAILED: Lingering scratch files found in working tree:"
  echo "$UNTRACKED"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Clean"
fi

# 4. Check for .env or private key files
echo -n "  Checking for staged secrets or .env files... "
STAGED_SECRETS=$(git diff --cached --name-only | grep -E "\.(env|pem|key|p12|p8)$" || true)
if [ -n "$STAGED_SECRETS" ]; then
  echo "🚨 CRITICAL: Staged credential files detected!"
  echo "$STAGED_SECRETS"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Clean"
fi

# 5. Summary
if [ "$FAILURES" -gt 0 ]; then
  echo "❌ Hygiene audit failed with $FAILURES issues. Resolve them before committing."
  exit 1
else
  echo "🎉 All hygiene checks passed cleanly. Ready for commit."
  exit 0
fi
```

---

## 5. Universal `.gitignore` Best Practices

Ensure every repository maintains a robust root `.gitignore` covering platform and IDE noise:

```gitignore
# macOS System Files
.DS_Store
.AppleDouble
.LSOverride
Icon?
._*

# Xcode & Apple Developer
DerivedData/
*.xcscmblueprint
*.xcuserstate
xcuserdata/
*.xcbkptlist
.build/
*.hmap
*.ipa
*.dSYM.zip
*.dSYM

# Node / TypeScript
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnpm-debug.log*
.next/
dist/
build/
.turbo/

# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
.venv/
.mypy_cache/
.ruff_cache/
.pytest_cache/

# Rust / Cargo
/target/
**/*.rs.bk

# Go
bin/
/pkg/

# Environment & Secrets
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.p8
*.mobileprovision
```

---

## 6. Pre-Flight Verification Checklist

Before marking any task complete or submitting a pull request, run through this checklist:

- [ ] **Working Tree**: `git status` shows zero unexpected untracked files or leftover scratch scripts.
- [ ] **Zero Warnings**: Compiler succeeds cleanly with warnings treated as errors.
- [ ] **Tests Green**: Entire test suite passes without flaky or disabled tests.
- [ ] **No Secret Leaks**: `git diff --cached` contains no API keys, private tokens, or credentials.
- [ ] **Atomic Commits**: Changes are grouped logically with Conventional Commit messages (`feat:`, `fix:`, `refactor:`, `test:`).
- [ ] **Documentation**: Any new commands, flags, or configuration options are updated in `README.md` or `AGENTS.md`.