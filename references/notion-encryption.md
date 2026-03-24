# Notion Encryption System

How secrets found during a security sweep are encrypted and stored in Notion.

## Overview

When the security sweep finds hardcoded secrets, you can encrypt them to your Notion workspace using `notion-secrets.js` — a local encryption tool that wraps secrets in AES-256-GCM before uploading to Notion.

**What Notion stores:** encrypted blobs — completely unreadable without the master password.
**What an attacker needs:** the master password + the Notion page ID.

## Architecture

```
Your machine                          Notion (cloud)
┌──────────────────────┐             ┌──────────────────────┐
│  Master Password      │             │                      │
│       ↓               │             │  Encrypted blob       │
│  PBKDF2 (100k iter)  │             │  [iv][ciphertext][tag]│
│       ↓               │             │                      │
│  AES-256-GCM encrypt │  ──push──>  │  Page: scan-results   │
│                      │             │                      │
│  AES-256-GCM decrypt │  ──pull──>  │  (Notion API)         │
│       ↓               │             │                      │
│  Plaintext secret     │             └──────────────────────┘
└──────────────────────┘
```

## Setup

### 1. Get a Notion API Integration

1. Go to [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Create a new integration ("Security Sweep")
3. Copy the internal API token
4. Share the secrets page/database with the integration

### 2. Store the integration token

```bash
# The token is stored encrypted — never in plain text
node ~/.openclaw/scripts/notion-secrets.js put notion_api_key "<your-token>"
```

### 3. Create a secrets database in Notion

Create a new page or database in Notion called "RhomBot Secrets" (or any name you prefer). Share it with your integration.

### 4. Set the master password

```bash
# The master password is derived into an encryption key — never stored
export NOTION_MASTER_PASSWORD="your-strong-master-password"
```

## Usage

```bash
# Encrypt and store
echo "$MASTER_PW" | node notion-secrets.js put <label> "<secret>"

# List (shows labels only, never values)
node notion-secrets.js list

# Decrypt (interactive, password never leaves your machine)
node notion-secrets.js decrypt "<blob>"
```

## How Encryption Works

**Algorithm:** AES-256-GCM
**Key derivation:** PBKDF2 with SHA-512, 100,000 iterations
**IV:** Random 16 bytes per encryption
**Tag:** 16 bytes (authentication tag)

```
plaintext → PBKDF2(pw, salt) → AES-256-GCM → [salt][iv][ciphertext][tag]
```

## Security Properties

| Property | Protection |
|----------|------------|
| Encrypted at rest | Notion only sees ciphertext |
| Key derivation | Brute-force resistant (100k PBKDF2 iterations) |
| Authenticated encryption | Tampering is detectable |
| No password storage | Master password never touches disk |
| Per-secret salt | Same secret encrypts differently each time |

## Limitations

- The master password must be entered each session (or stored in your shell profile's env)
- Notion API rate limits apply (~3 requests/second)
- If Notion is down, secrets are unavailable (download and decrypt locally if you need offline access)

## For CI/CD

In CI environments, inject the master password as a secret environment variable:

```bash
export NOTION_MASTER_PASSWORD="$NOTION_MASTER_PASSWORD"
bash scripts/full-scan.sh --encrypt-found [options]
```

The password never appears in logs — only the "Stored" confirmation is visible.
