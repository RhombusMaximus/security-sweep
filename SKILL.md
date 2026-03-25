---
name: security-sweep
description: Security scanner for OpenClaw skills and plugins. Scans for hardcoded secrets, dangerous exec patterns, dependency vulnerabilities, and network egress. When secrets are found, auto-encrypts them to Notion using the user's encrypted secrets store. Use when auditing skills/plugins, before publishing to ClawHub or GitHub, or when a user requests a security review.
version: 1.0.1
---

# Security Sweep — Skill & Plugin Auditor

Scans OpenClaw skills and plugins for:
1. **Hardcoded secrets** — API keys, tokens, passwords in code
2. **Dangerous exec patterns** — shell injection, eval, unsanitized child_process calls
3. **Dependency vulnerabilities** — npm audit failures
4. **Network egress** — unexpected outbound connections
5. **Input injection** — unsanitized user input reaching exec/file/eval

When secrets are found, they are **auto-encrypted to Notion** using the user's encrypted secrets store — never stored in plain text.

## Scan Scope

**Built-in skills** (read-only, bundled with OpenClaw CLI):
```
$(brew --prefix)/Cellar/openclaw-cli/<version>/libexec/lib/node_modules/openclaw/skills/
```

**Workspace skills** (user-installed):
```
~/.openclaw/workspace/skills/
```

## Workflow

### Full Sweep (recommended)

```bash
SKILLS_DIR="$(brew --prefix)/Cellar/openclaw-cli/<version>/libexec/lib/node_modules/openclaw/skills"
WS_DIR="$HOME/.openclaw/workspace/skills"
REPORT_DATE=$(date +%Y%m%d_%H%M%S)
REPORT_FILE="$HOME/.openclaw/security-sweep-${REPORT_DATE}.txt"

bash ~/.openclaw/workspace/skills/security-sweep/scripts/full-scan.sh \
  --builtin "$SKILLS_DIR" \
  --workspace "$WS_DIR" \
  --output "$REPORT_FILE"
```

### Quick Scan (fast patterns only, no npm audit)
```bash
bash ~/.openclaw/workspace/skills/security-sweep/scripts/quick-scan.sh \
  --dir "$HOME/.openclaw/workspace/skills"
```

### Single Skill Scan
```bash
bash ~/.openclaw/workspace/skills/security-sweep/scripts/skill-scan.sh \
  --skill /path/to/skill
```

## When Secrets Are Found

> ⚠️ **WARNING:** `--encrypt-found` encrypts secrets to Notion before removal. This assumes your private skill directory is already secure. If your private skills already contain uncommitted real credentials, encrypting them to Notion does not retroactively protect those already-exposed secrets. Review findings manually before using this flag on an untrusted or already-compromised skillbase.
>
> **⚠️ CRITICAL:** The master password is the only decryption key. **Forget it and the secrets are permanently unrecoverable** — AES-256-GCM with PBKDF2 (100k iterations) is intentionally slow and cannot be brute-forced. Store it in a password manager.

The scan detects potential secrets and can encrypt them to your Notion secrets store:

```bash
# Store a secret — use NOTION_MASTER_PASSWORD env var (required for reliable scripting)
NOTION_MASTER_PASSWORD="your-password" node ~/.openclaw/scripts/notion-secrets.js put <label> "<secret>"

# List all stored secrets (names only, no password needed)
node ~/.openclaw/scripts/notion-secrets.js list

# Get a secret's encrypted blob (no password needed to retrieve)
node ~/.openclaw/scripts/notion-secrets.js get <label>

# Decrypt a blob (interactive prompt)
node ~/.openclaw/scripts/notion-secrets.js decrypt "<blob>"
```

**Encryption:** AES-256-GCM with PBKDF2 key derivation (100k iterations). Notion stores only encrypted blobs — useless without the master password.

See `references/notion-encryption.md` for setup instructions.

## Risk Categories

| Level | Finding | Action |
|-------|---------|--------|
| 🔴 CRITICAL | Hardcoded secret (api_key, token, password) | Remove immediately, encrypt to Notion, rotate credential |
| 🔴 CRITICAL | `eval()` on untrusted input | Replace with safe alternative |
| 🟠 HIGH | `exec()`, `spawn()` with string concatenation | Use execFile with array args |
| 🟠 HIGH | Shell injection surface (bash -c, `${var}` in shell) | Sanitize or use execFile |
| 🟡 MEDIUM | npm audit findings (any severity) | Review and update dependencies |
| 🟡 MEDIUM | Unexpected network egress | Verify necessity, document purpose |
| 🟢 LOW | File permission too broad (0o777) | Restrict to 0o644/0o755 |
| 🟢 INFO | process.env leak in logs | Ensure logs redact env vars |

## Before Publishing (GitHub / ClawHub)

1. Run full sweep
2. Fix all CRITICAL/HIGH findings
3. Verify no secrets in any file
4. Confirm npm audit passes with 0 vulnerabilities
5. Document all required env vars in SKILL.md

## Periodic Scanning

Schedule weekly sweeps via cron:
```bash
openclaw cron add \
  --name "security-sweep" \
  --every 604800 \
  --sessionTarget isolated \
  --payload '{"kind":"agentTurn","message":"Run security sweep on all skills. Report findings. Save report to ~/.openclaw/security-sweep-<date>.txt."}'
```

## Notion Encryption Setup

If you want the auto-encrypt feature:

1. Copy `references/notion-encryption.md` to understand the system
2. Create a Notion page/database to store encrypted secrets
3. Add your Notion API key to the secrets store:
   ```bash
   node ~/.openclaw/scripts/notion-secrets.js put notion_api_key "<your-key>"
   ```
4. Use the encrypt-found script when secrets are detected

## Sharing / Publishing

This skill is designed to be published to ClawHub or open-sourced on GitHub. Before publishing:
- [ ] Run full sweep with zero CRITICAL/HIGH findings
- [ ] All npm audits pass
- [ ] No hardcoded secrets anywhere
- [ ] README.md is complete and accurate
- [ ] LICENSE is included (MIT)
