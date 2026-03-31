# Security Sweep for OpenClaw

A comprehensive security scanner for OpenClaw skills and plugins. Scans for hardcoded secrets, dangerous exec patterns, dependency vulnerabilities, network egress, and shell injection surfaces.

> **📝 Preferred secrets store: 1Password** — Use 1Password as your primary secrets store. It provides audit logs, access controls, and service account tokens for automation. See the 1Password skill for setup. The Notion encrypted store is deprecated.

---

## Features

- 🔍 **Secret Detection** — Scans for API keys, tokens, passwords, credentials hardcoded in skill files
- 🐚 **Exec Pattern Analysis** — Finds dangerous `exec()`, `spawn()`, `eval()`, and shell injection surfaces
- 📦 **NPM Audit** — Checks for vulnerable dependencies in skills with `package.json`
- 🌐 **Network Egress** — Reports unexpected outbound connections
- 🔒 **Secrets Store: 1Password** — Found secrets should be stored in 1Password (preferred) or a dedicated secrets manager — never in plaintext files or transcripts
- 🚀 **CI-Ready** — Exit codes and structured output for use in GitHub Actions

---

## Quick Start

```bash
# Scan all skills
bash scripts/full-scan.sh \
  --builtin "$(brew --prefix)/Cellar/openclaw-cli/2026.3.13/libexec/lib/node_modules/openclaw/skills" \
  --workspace "$HOME/.openclaw/workspace/skills" \
  --output ~/security-sweep-report.txt

# Quick scan (fast patterns only)
bash scripts/quick-scan.sh --dir ~/.openclaw/workspace/skills
```

---

## Reports

Reports are saved to the output file you specify. Each finding includes:
- File path
- Risk level (CRITICAL / HIGH / MEDIUM / LOW / INFO)
- Recommended action

Sample output:
```
━━━ WORKSPACE SKILLS Summary ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🔴 CRITICAL: 0
  🟠 HIGH:     0
  🟡 MEDIUM:   0
  🟢 LOW:      0
  ℹ️  INFO:     3

━━━ Findings ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ℹ️  NETWORK: skill/example/public/js/api.js
  ℹ️  NETWORK: skill/example/src/index.js
```

---

## Risk Levels

| Level | Description | Example |
|-------|-------------|---------|
| 🔴 CRITICAL | Hardcoded secret or unsafe eval | `const key = "sk-123..."` |
| 🟠 HIGH | Dangerous exec without sanitization | `exec(userInput)` |
| 🟡 MEDIUM | Shell injection surface or npm audit failure | `bash -c "${var}"` |
| 🟢 LOW | Overly broad file permissions | `chmod 777` |
| ℹ️ INFO | Expected network egress documented | External API call |

---

## ⚠️ Warning: Found Secrets Must Go to 1Password

When secrets are found during a scan, they must be stored in 1Password — not in plaintext files, chat, or shell history.

**Best practice:**
1. Add the secret to 1Password immediately
2. If the secret is a leaked/rotated credential, invalidate the old one first
3. Never commit secrets to git, even "temporarily"

**If scanning a skill that needs a secret:** Document the required env vars in SKILL.md. The skill should pull secrets from 1Password at runtime, not store them locally.

## Secrets Storage: 1Password (Preferred)

Found secrets should be encrypted and stored in 1Password. This requires:
1. `op` CLI installed and authenticated (see 1Password skill)
2. The secret added to your vault before removal from code

```bash
# Add to vault
op item create --category="API Credential" --title="My Service Key" \
  --vault "My Vault" "credential=$SECRET_VALUE"
```

The security sweep does NOT auto-encrypt — it flags findings for manual remediation. This is intentional: auto-encryption without review can cause data loss.

---

## Before Publishing

Run a full scan and confirm:
- [ ] 0 CRITICAL findings
- [ ] 0 HIGH findings
- [ ] All npm audits pass
- [ ] No secrets anywhere in the codebase
- [ ] All network egress is expected and documented

---

## GitHub Actions Integration

```yaml
- name: Security Sweep
  run: |
    bash scripts/full-scan.sh \
      --builtin "./skills" \
      --output "./security-report.txt"
    # Fail on CRITICAL or HIGH
    grep -qE "🔴|🟠" ./security-report.txt && exit 1
```

---

## License

MIT — free to use, modify, and distribute.
