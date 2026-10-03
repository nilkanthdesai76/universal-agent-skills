# CODE_STYLE.md — Engineering Standards & Style Guide

> This document establishes formatting rules, naming conventions, and language idioms.

---

## 🎨 General Principles

- **Readability over Cleverness**: Explicit is better than implicit.
- **Strict Concurrency & Immutability**: Prefer immutable `let` / `const` bindings; use Sendable types and avoid mutable shared state.
- **Fail Fast & Explicit Errors**: Prefer typed errors and explicit pattern matching over silent default fallbacks.

---

## 📐 Naming Conventions

- **Types & Protocols**: `UpperCamelCase` (e.g. `NetworkClient`, `EndpointProvider`).
- **Methods & Variables**: `lowerCamelCase` (e.g. `fetchUserProfile()`, `maximumRetryCount`).
- **Constants & Enums**: `lowerCamelCase` enum cases (e.g. `.notFound`, `.timeout`).

---

## 🧹 Linters & Formatters

```bash
# SwiftFormat / SwiftLint
swiftformat .
swiftlint --fix

# Prettier / ESLint
npm run lint -- --fix
```
