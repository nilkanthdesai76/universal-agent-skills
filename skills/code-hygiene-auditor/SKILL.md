---
name: code-hygiene-auditor
description: >-
  Performs pre-commit quality and security audits: scans for hardcoded API keys/secrets, checks for unhandled errors or compiler warnings, and verifies clean working tree state.
---

# Code Hygiene Auditor Skill

Run this audit skill before committing changes or concluding user requests.

## Audit Checklist

1. **Secret Scanning**:
   - Verify zero API keys (`ghp_`, `sk_`, private tokens, passwords, bearer tokens) exist in any tracked files.

2. **Sanitize Git Config**:
   - Ensure local `.git/config` remote URLs do not contain embedded access tokens.

3. **Compiler Warnings & Strict Concurrency**:
   - Check that builds pass with zero errors and no concurrency data-race warnings.

4. **Multi-Commit Hygiene**:
   - Avoid massive monolithic 1-commit dumps. Structure commits logically: chore -> feat -> test -> docs.
