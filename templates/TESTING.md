# TESTING.md — Testing Standards & Verification Playbook

> This document defines testing pyramids, mock strategies, and execution commands.
> Agents MUST verify changes using these procedures before declaring tasks complete.

---

## 🧪 Testing Philosophy

1. **Deterministic Tests**: No reliance on live third-party network APIs or unmocked real timers.
2. **Fast Feedback**: Unit tests should execute in `< 5 seconds` locally.
3. **High Regression Safety**: Every bug fix MUST include a reproduction test.

---

## 🏃 Running Tests

```bash
# Run all unit tests
swift test # or: npm test / pytest

# Run with verbose output
swift test -v

# Run targeted test method
swift test --filter <TestSuiteName>.<TestMethodName>
```

---

## 🎭 Mocking & Dependency Injection

- Use protocol-based mocks rather than subclassing real production classes.
- Place test helpers and mock objects in dedicated `Tests/<Target>Tests/Mocks/` directories.
