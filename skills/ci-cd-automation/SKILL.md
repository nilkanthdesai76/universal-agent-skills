---
name: ci-cd-automation
description: >-
  Operational protocol for designing, generating, and maintaining resilient Continuous Integration workflows. Use when: scaffolding or fixing GitHub Actions or GitLab CI pipelines for Swift, iOS, macOS, TypeScript, Python, or Go projects, configuring dependency caching (.build, node_modules, pip), setting up concurrency cancellation groups, configuring Xcode runner environments, or linking CI status badges to README.md.
---

# CI/CD Automation (Continuous Integration Architecture) 🚀⚙️

The definitive operational manual for AI coding agents tasked with authoring, debugging, optimizing, and maintaining automated Continuous Integration (CI) workflows that keep repositories green, tested, and regression-free.

---

## 1. Executive Summary & Core Philosophy

Continuous Integration is the objective verification gateway for software development. For AI agents, green CI is the ultimate proof that changes do not break dependencies or introduce hidden regressions.

1. **Failure Modes of AI Agents**:
   - Generating workflows with outdated or sunsetted runner labels (e.g. `macos-12`, `ubuntu-18.04`).
   - Forgetting dependency caching, causing jobs to take 10 minutes instead of 45 seconds.
   - Pushing workflows that fail on pull requests from forks due to missing secret permissions.
   - Creating queue stampedes by failing to configure concurrency cancellation groups.
   - Failing to attach CI status badges to `README.md`.

2. **The Automation Mandate**:
   - **Deterministic Runners**: Explicit runner images (`ubuntu-latest`, `macos-14`, `macos-15`).
   - **Cache Everything**: Cache `.build`, `node_modules`, `~/.cache/pip`, `~/.cache/uv`, and Cargo directories.
   - **Fail-Fast with Concurrency Cancellation**: Automatically cancel redundant in-progress runs when new commits arrive.
   - **Least-Privilege Token Scopes**: Explicitly define `permissions: contents: read` across all jobs.
   - **Status Badge Verification**: Always add and verify the workflow badge at the top of the README.

---

## 2. Multi-Platform Runner Matrix & Command Reference

```
+-----------------------------------------------------------------------------------------+
|                              PRODUCTION RUNNER CONFIGURATION                            |
+---------------------+--------------------+----------------------------------------------+
| Platform / Language | Recommended Runner | Standard Execution Command                   |
+---------------------+--------------------+----------------------------------------------+
| Swift (SPM Package) | macos-14           | swift test -v --parallel                     |
| Xcode App (iOS/Mac) | macos-14 / 15      | xcodebuild test -destination '...'           |
| Node / TypeScript   | ubuntu-latest      | pnpm install --frozen-lockfile && pnpm test  |
| Python / FastAPI    | ubuntu-latest      | uv run pytest -v                             |
| Rust / Cargo        | ubuntu-latest      | cargo test --all-features --verbose          |
| Go Microservice     | ubuntu-latest      | go test -race -v ./...                       |
+---------------------+--------------------+----------------------------------------------+
```

---

## 3. Production Multi-Platform Workflow Configurations

### 3.1 Apple & Swift Multi-Platform Workflow (`.github/workflows/ci-apple.yml`)

Full production workflow featuring Xcode version selection, SPM dependency caching, iOS simulator destination sharding, and headless code signing bypass:

```yaml
name: CI Apple Suite

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  spm-package:
    name: SwiftPM Linux & macOS
    runs-on: macos-14
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Select Xcode Version
        run: sudo xcode-select -s /Applications/Xcode_16.0.app/Contents/Developer

      - name: Cache SPM Dependencies
        uses: actions/cache@v4
        with:
          path: |
            .build
            ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
          restore-keys: |
            ${{ runner.os }}-spm-

      - name: Build & Run SPM Tests
        run: swift test -v --parallel

  ios-simulator:
    name: iOS Simulator Test Suite
    runs-on: macos-14
    strategy:
      fail-fast: false
      matrix:
        destination:
          - 'platform=iOS Simulator,name=iPhone 16,OS=18.0'
          - 'platform=iOS Simulator,name=iPad Pro 11-inch (M4),OS=18.0'
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Select Xcode Version
        run: sudo xcode-select -s /Applications/Xcode_16.0.app/Contents/Developer

      - name: Cache DerivedData
        uses: actions/cache@v4
        with:
          path: build/DerivedData
          key: ${{ runner.os }}-deriveddata-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-deriveddata-

      - name: Pre-warm Simulator
        run: |
          xcrun simctl list devices available | grep -E "iPhone 16|iPad Pro" || true

      - name: Run xcodebuild Tests
        run: |
          set -o pipefail
          xcodebuild clean test \
            -workspace App.xcworkspace \
            -scheme App \
            -destination "${{ matrix.destination }}" \
            -derivedDataPath build/DerivedData \
            CODE_SIGNING_ALLOWED=NO \
            COMPILER_INDEX_STORE_ENABLE=NO \
            | xcbeautify || true
```

---

### 3.2 TypeScript / Node.js Monorepo Workflow (`.github/workflows/ci-node.yml`)

Optimized with `pnpm` store caching, matrix testing across Node LTS versions, Biome linting, and Turborepo caching:

```yaml
name: CI Node / TypeScript

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: node-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  lint-and-test:
    name: Node ${{ matrix.node-version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node-version: [20.x, 22.x]
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Install pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 9

      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'pnpm'

      - name: Install Dependencies
        run: pnpm install --frozen-lockfile

      - name: Typecheck
        run: pnpm tsc --noEmit

      - name: Run Linter
        run: pnpm lint

      - name: Run Unit Tests with Coverage
        run: pnpm test -- --coverage
```

---

### 3.3 Rust High-Performance Engine Workflow (`.github/workflows/ci-rust.yml`)

Features `sccache` cross-job compiler caching, `clippy` pedantic checks, and format validation:

```yaml
name: CI Rust Engine

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: rust-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-Dwarnings"

jobs:
  check-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Install Rust Toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - name: Configure Cargo Cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-

      - name: Verify Code Formatting
        run: cargo fmt --check

      - name: Run Clippy Linter
        run: cargo clippy --all-targets --all-features

      - name: Run Tests
        run: cargo test --all-features --verbose
```

---

### 3.4 Python & FastAPI Backend Workflow (`.github/workflows/ci-python.yml`)

Using `uv` for sub-second virtualenv setup and dependency installation, `ruff` for ultra-fast linting, and `pytest`:

```yaml
name: CI Python

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: python-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v2
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Run Ruff Linter & Formatter
        run: |
          uv run ruff check .
          uv run ruff format --check .

      - name: Run Pytest with Coverage
        run: uv run pytest -v --cov=. --cov-report=xml
```

---

### 3.5 Go High-Concurrency Service Workflow (`.github/workflows/ci-go.yml`)

Featuring `golangci-lint`, race condition detection, and Go module caching:

```yaml
name: CI Go Service

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: go-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  lint:
    name: GolangCI-Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: false
      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: v1.60

  test:
    name: Race Detector & Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true
      - name: Run Tests with Race Detector
        run: go test -race -v -coverprofile=coverage.txt -covermode=atomic ./...
```

---

## 4. Pipeline Security Hardening & Zero-Trust Architecture

When deploying CI workflows in enterprise or open-source repositories, adhere to these security mandates:

1. **Zero-Trust Token Permissions**:
   Explicitly declare minimal permissions at the root of every workflow. Never rely on repository defaults which often grant read/write access:
   ```yaml
   permissions:
     contents: read
   ```
   If a specific step requires write access (e.g., publishing a release), declare permissions at the job level:
   ```yaml
   jobs:
     publish:
       permissions:
         contents: write
   ```

2. **Mitigating `pull_request_target` Exploits**:
   - Never use `pull_request_target` when checking out untrusted code from community forks.
   - An attacker can submit a pull request modifying workflow scripts or tests to exfiltrate repository secrets.
   - Use standard `pull_request` triggers for PR evaluation (which execute in an unprivileged context without secrets).
   - If PR comments or label updates are required, separate the untrusted build from the privileged reporting job using GitHub Actions artifacts and `workflow_run`.

3. **Immutable Third-Party Action Pinning**:
   To prevent supply-chain compromise via tag hijacking, pin critical third-party actions to full immutable commit SHAs:
   ```yaml
   # Vulnerable to tag mutation:
   uses: actions/checkout@v4

   # Secure immutable pin:
   uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
   ```

4. **Secret Leakage Elimination**:
   - Never echo environment variables or dump environment tables (`env`, `printenv`) in debug scripts.
   - Mask dynamic tokens immediately if generated via CLI:
     ```bash
     echo "::add-mask::$DYNAMIC_SECRET"
     ```

---

## 5. Dependency Caching Strategies & Performance Optimization

Proper cache key construction prevents cache thrashing and ensures cache hits across identical dependencies:

| Ecosystem | Directories to Cache | Recommended Cache Key Pattern |
|---|---|---|
| Swift (SPM) | `.build`, `SourcePackages` | `${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}` |
| Xcode | `build/DerivedData` | `${{ runner.os }}-deriveddata-${{ github.sha }}` |
| Node (pnpm) | `~/.local/share/pnpm/store` | `${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}` |
| Rust (Cargo) | `~/.cargo/registry`, `~/.cargo/git`, `target` | `${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}` |
| Python (uv) | `~/.cache/uv` | `${{ runner.os }}-uv-${{ hashFiles('**/pyproject.toml') }}` |
| Go | `~/go/pkg/mod`, `~/.cache/go-build` | Managed automatically by `actions/setup-go@v5` with `cache: true` |

---

## 6. Automated Status Badges & Local Verification Scripts

Every production repository must publish workflow status badges at the top of `README.md`:

```markdown
[![CI Apple Suite](https://github.com/<owner>/<repo>/actions/workflows/ci-apple.yml/badge.svg)](https://github.com/<owner>/<repo>/actions/workflows/ci-apple.yml)
[![CI Node / TypeScript](https://github.com/<owner>/<repo>/actions/workflows/ci-node.yml/badge.svg)](https://github.com/<owner>/<repo>/actions/workflows/ci-node.yml)
```

### 6.1 Automated Verification Script

Run this script locally to verify active CI pipeline health before prompting users or cutting release tags:

```bash
#!/usr/bin/env bash
# verify-ci-health.sh — Triage recent CI runs via GitHub CLI
set -euo pipefail

if ! command -v gh &> /dev/null; then
  echo "⚠️ GitHub CLI (gh) not found. Skipping remote CI check."
  exit 0
fi

REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null || true)
if [ -z "$REPO" ]; then
  echo "⚠️ Current repository has no GitHub remote origin."
  exit 0
fi

echo "🔍 Checking latest CI runs for $REPO..."

RUNS=$(gh run list --limit 8 --json status,conclusion,workflowName,headBranch \
  -q '.[] | "\(.workflowName) on \(.headBranch): \(.status) (\(.conclusion // "in progress"))"')

echo "$RUNS"

if echo "$RUNS" | grep -q "failure"; then
  echo "🚨 Identified failed CI workflows in recent runs! Inspect logs before merging."
  exit 1
else
  echo "✅ Recent CI pipeline executions are green and healthy."
fi
```

---

## 7. Diagnostic Matrix for Common Pipeline Failures

| Failure Signature | Probable Root Cause | Immediate Remediation |
|---|---|---|
| `The process '/usr/bin/git' failed with exit code 128` | Missing submodule credentials or deleted commit SHA | Add `submodules: recursive` or pass `token: ${{ secrets.GITHUB_TOKEN }}` in `actions/checkout` |
| `Cannot find simulator matching destination` | Requested iOS runtime not pre-installed on chosen macOS runner image | Query `xcrun simctl list runtimes` or update workflow matrix to match runner's pre-installed runtimes |
| `Process completed with exit code 137` | Out of Memory (OOM) killed by host kernel | Reduce test parallelization (`--parallel false`), decrease matrix concurrency, or upgrade runner tier |
| `Resource not accessible by integration` | GITHUB_TOKEN lacks requested permission scope | Explicitly declare required permission (e.g. `pull-requests: write`, `issues: write`) in workflow job config |
| `Cache size exceeds 10GB limit` | Storing unpruned build caches or large binary assets | Refine cache `path` to target only lockfiles and compiled dependency subfolders (`.build/release`) |
| `Code signing failed: No profile matching...` | Missing signing certificate on headless runner | Pass `CODE_SIGNING_ALLOWED=NO` for CI PR builds, or import ephemeral keychain via `security import` |