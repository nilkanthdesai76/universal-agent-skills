---
name: zero-secret-sanitizer
description: >-
  Operational protocol for scanning, detecting, and remediating leaked secrets and credentials. Use when: scanning working directories or staged changes for hardcoded credentials (OpenAI, GitHub, AWS, Stripe, database passwords, private keys), generating safe .env.example templates, stripping personal access tokens from .git/config remote URLs, or scrubbing accidental secrets from git history before public release.
---

# Zero-Secret Sanitizer (Credential Protection & History Purification) 🔒🛡️

The definitive operational manual for AI coding agents tasked with preventing credential leaks, auditing working directories, generating clean `.env.example` templates, and scrubbing contaminated git histories.

---

## 1. Executive Summary & Core Philosophy

A single leaked API key, database password, or private certificate committed to a git repository can lead to immediate infrastructure compromise through automated internet-wide scanning bots within seconds of being pushed. AI coding assistants are uniquely susceptible to accidental leaks because they often create test scripts, copy staging tokens into `.env`, or embed personal access tokens into git remote URLs.

1. **Failure Modes of AI Agents**:
   - Hardcoding personal access tokens (`ghp_...`), OpenAI keys (`sk-...`), or database connection strings directly into test scripts.
   - Pushing active `.env` files because `.gitignore` lacked `.env` entries.
   - Embedding GitHub PATs in `.git/config` remote URLs (`https://user:ghp_xxx@github.com/...`).
   - Checking in development `.pem`, `.key`, or `.p12` certificates.
   - Failing to rotate a compromised token immediately after accidental exposure.

2. **The Sanitizer's Mandate**:
   - **Zero Secrets in Tracked Files**: Scan every staged change (`git diff --cached`) before committing.
   - **Mandatory `.env.example`**: Never commit credentials; always supply a template with placeholder values (`YOUR_API_KEY_HERE`).
   - **Sanitized Git Remotes**: Remote URLs must use standard SSH (`git@github.com:...`) or system credential managers.
   - **Immediate Revocation**: If a token is committed or pasted into chat, treat it as compromised and rotate immediately.

---

## 2. High-Entropy Secret Detection Patterns (Regex Compendium)

Use these regular expressions to audit codebases and staging buffers:

```regex
# 1. GitHub Personal Access Tokens:
Classic PAT:         ghp_[a-zA-Z0-9]{36}
Fine-Grained PAT:    github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}
OAuth Access Token:  gho_[a-zA-Z0-9]{36}
User-to-Server:      ghu_[a-zA-Z0-9]{36}

# 2. AI Providers:
OpenAI API Key:      sk-[a-zA-Z0-9]{32,48}
OpenAI Project Key:  sk-proj-[a-zA-Z0-9_-]{50,}
Anthropic API Key:   sk-ant-[a-zA-Z0-9_-]{80,}

# 3. Cloud Infrastructure:
AWS Access Key ID:   AKIA[0-9A-Z]{16}
AWS Secret Key:      (?i)aws_secret_access_key\s*=\s*[a-zA-Z0-9/+=]{40}
Google Cloud API:    AIza[0-9A-Za-z\\-_]{35}

# 4. Payment Gateways:
Stripe Secret Key:   sk_live_[0-9a-zA-Z]{24,34}
Stripe Restricted:   rk_live_[0-9a-zA-Z]{24,34}

# 5. Database Connection Strings:
PostgreSQL URI:      postgres(?:ql)?:\/\/[a-zA-Z0-9_]+:[^@\s]+@[a-zA-Z0-9.-]+:[0-9]+\/[a-zA-Z0-9_-]+
MongoDB URI:         mongodb(?:\+srv)?:\/\/[a-zA-Z0-9_]+:[^@\s]+@[a-zA-Z0-9.-]+\/
Redis URI:           redis:\/\/(?::[^@\s]+@)?[a-zA-Z0-9.-]+:[0-9]+

# 6. Private Cryptographic Keys:
RSA Private Key:     -----BEGIN RSA PRIVATE KEY-----
EC Private Key:      -----BEGIN EC PRIVATE KEY-----
OpenSSH Private:     -----BEGIN OPENSSH PRIVATE KEY-----
Generic Private:     -----BEGIN PRIVATE KEY-----
```

---

## 3. Production Pre-Commit Secret Scanner Script

Deploy this automated scanner in `scripts/scan-secrets.sh` to enforce zero-secret commitments:

```bash
#!/usr/bin/env bash
# scan-secrets.sh — Audits staged files for credential patterns
set -euo pipefail

echo "🔒 Starting Zero-Secret Security Audit..."

# Patterns to scan for
PATTERNS=(
  "ghp_[a-zA-Z0-9]{36}"
  "github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}"
  "sk-[a-zA-Z0-9]{32,}"
  "sk-proj-[a-zA-Z0-9_-]{30,}"
  "sk-ant-[a-zA-Z0-9_-]{30,}"
  "AKIA[0-9A-Z]{16}"
  "sk_live_[0-9a-zA-Z]{24}"
  "-----BEGIN.*PRIVATE KEY-----"
  "postgres:\/\/.*:.*@"
  "mongodb.*:\/\/.*:.*@"
)

FINDINGS=0

# Check staged git diff
for pattern in "${PATTERNS[@]}"; do
  MATCHES=$(git diff --cached -G "$pattern" --name-only 2>/dev/null || true)
  if [ -n "$MATCHES" ]; then
    echo "🚨 DETECTED SECRET matching pattern '$pattern' in staged files:"
    echo "$MATCHES" | sed 's/^/   • /'
    FINDINGS=$((FINDINGS + 1))
  fi
done

# Check untracked .env files staged by mistake
STAGED_ENV=$(git diff --cached --name-only | grep -E "(^|\/)\.env(\..+)?$" | grep -v "\.env\.example" || true)
if [ -n "$STAGED_ENV" ]; then
  echo "🚨 CRITICAL: Staged active environment file:"
  echo "$STAGED_ENV" | sed 's/^/   • /'
  FINDINGS=$((FINDINGS + 1))
fi

if [ "$FINDINGS" -gt 0 ]; then
  echo "❌ Security audit FAILED with $FINDINGS secret violations. Unstage and sanitize before committing."
  exit 1
else
  echo "✅ Zero secrets detected in staged files. Safe to commit."
  exit 0
fi
```

---

## 4. Git Remote Credential Sanitization Protocol

Agents often clone repositories using embedded tokens in the URL. Check and sanitize `.git/config`:

```bash
# Check remote URL for embedded credentials:
git remote get-url origin
# If output contains https://user:ghp_xxx@github.com/...:

# 1. Strip the token and revert to clean HTTPS:
REPO_PATH=$(git remote get-url origin | sed -E 's/https:\/\/[^@]+@github\.com\//https:\/\/github.com\//')
git remote set-url origin "$REPO_PATH"

# Or switch to SSH:
git remote set-url origin "git@github.com:<owner>/<repo>.git"

# 2. Verify git remote is clean:
git remote -v
```

---

## 5. Emergency Git History Scrubbing (`git filter-repo`)

If a secret was accidentally committed into git history:

### Step 1: Immediate Token Revocation
Immediately visit the provider dashboard (GitHub, OpenAI, AWS, Stripe) and **revoke the token**. Once a secret touches git history, assume it is compromised regardless of whether the commit was pushed.

### Step 2: Scrubbing Using `git-filter-repo` (Recommended Modern Tool)

```bash
# Install git-filter-repo if not already available
pip install git-filter-repo

# Scrub a specific file containing secrets from all commits:
git filter-repo --path .env --invert-paths --force

# Replace a specific token string with a placeholder across all commits:
echo "ghp_compromised_token==>REDACTED_TOKEN" > /tmp/replace.txt
git filter-repo --replace-text /tmp/replace.txt --force
rm /tmp/replace.txt

# Force push scrubbed history (ensure collaborators are notified):
git push origin main --force
```

### Step 3: Lightweight Remediation for Unpushed Commits
If the commit has **NOT** yet been pushed to any remote:

```bash
# Undo commit while preserving modified files in working tree
git reset --soft HEAD~1

# Remove the secret from the file and replace with placeholder
# Ensure .env is added to .gitignore
echo ".env" >> .gitignore
git add .gitignore

# Stage sanitized files and recommit cleanly
git commit -m "chore(config): add .env to gitignore and sanitize template"
```

---

## 6. Safe Configuration Architecture (`.env.example`)

Every project requiring environment configuration must commit a sanitized template:

```env
# .env.example — Application Environment Configuration Template
# Copy this file to .env and provide local values. Do NOT commit .env to git.

# Server Configuration
PORT=8080
NODE_ENV=development

# Database
DATABASE_URL=postgres://postgres:password@localhost:5432/mydb

# External APIs
OPENAI_API_KEY=YOUR_OPENAI_API_KEY_HERE
ANTHROPIC_API_KEY=YOUR_ANTHROPIC_API_KEY_HERE
GITHUB_TOKEN=YOUR_GITHUB_PERSONAL_ACCESS_TOKEN_HERE

# Authentication Secrets
JWT_SECRET=generate_with_openssl_rand_base64_32
```

---

## 7. Pre-Commit Security Checklist

Before finalizing any task or pushing to remote repositories:

- [ ] **Staged Diff Inspected**: Ran `git diff --cached` to confirm zero live credentials or private keys.
- [ ] **.gitignore Configured**: Verified that `.env`, `.env.local`, `*.pem`, `*.key`, and `*.p12` are listed.
- [ ] **Remote URL Clean**: Verified `git remote get-url origin` contains no embedded access tokens.
- [ ] **Safe Placeholders**: Configuration examples use obvious placeholders (`YOUR_API_KEY_HERE`).
- [ ] **Tokens Rotated**: If any real credential was exposed during the session, it was rotated in the provider console immediately.