---
name: ci-cd-automation
description: >-
  Scaffolds automated GitHub Actions CI/CD workflows for Swift, TypeScript/Node, Python, and Go projects with dependency caching, testing, linting, and build verification badges.
---

# CI/CD Automation Skill

Guides agents in setting up continuous integration workflows that keep projects clean, reliable, and regression-free.

## Best Practices

1. **Deterministic Environment Runners**:
   - For Apple/Swift projects: use `macos-14` or `macos-15` runners with Xcode tooling.
   - For Web/Node/Python: use `ubuntu-latest` with explicit language versions.

2. **Standard Workflow Template**:
   - Checkout repo (`actions/checkout@v4`).
   - Run linter/typecheck.
   - Run unit tests with verbose diagnostics.

3. **Status Badges**:
   - Always attach the generated CI status badge to the top of `README.md`.
