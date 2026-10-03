# SECURITY.md — Security Guidelines & Threat Model

> This document defines secret prevention rules, access controls, and vulnerability reporting.

---

## 🔒 Zero Secrets Policy

1. **No Hardcoded Secrets**: Under NO circumstances should API keys, bearer tokens, passwords, private SSH keys, or staging credentials be hardcoded in source code or committed to git.
2. **Environment Variables**: All secret configuration must be loaded via secure environment variables or Keychain.
3. **Template Environment Files**: Maintain a sanitized `.env.example` file showing required variable names with placeholder values.

---

## 🛡️ Input Validation & Sanitization

- Sanitize all external inputs, URL parameters, and user-generated text.
- Prevent SQL injection, command injection, and cross-site scripting (XSS).
- Keep third-party dependencies updated and scan for known CVEs.
