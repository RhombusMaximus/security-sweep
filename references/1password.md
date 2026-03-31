# Secrets Storage: 1Password for Security Sweeps

**Preferred secrets store: 1Password.** Use it for all secrets found during security sweeps or any other context.

## Why 1Password over Notion

- **Audit logs** — 1Password records every access to vault items
- **Service accounts** — automation without interactive login (WSL2 compatible)
- **No encryption complexity** — secrets stored directly, not as encrypted blobs
- **Client apps** — iOS/Android for quick access; browser extension; CLI
- **Audit trail** — every access logged per user/account

## Architecture

```
┌─────────────────┐    op CLI     ┌─────────────────────┐
│  Your machine   │ ←───────────→ │  1Password Cloud    │
│                 │   service      │  (vault + audit)     │
│  op CLI / SDK   │   account      │                     │
└─────────────────┘               └─────────────────────┘
```

## Setup

### 1. Create a service account (recommended for automation)

1. Go to your 1Password account → **Settings → Security → Service Accounts**
2. Create a new service account with appropriate vault permissions
3. Copy the token — treat it like any other secret

### 2. Store token on this machine

```bash
# Add to 1Password vault first:
op item create --category="API Credential" --title="1Password Service Account Token" \
  --vault "RhomBot's Vault" "credential=<token>"

# Then source it for scripts:
source /home/openclaw/.openclaw/op-env.sh && op vault list
```

### 3. Persist via systemd (for gateway/daemon use)

```bash
# Token stored in op-service.env, loaded via systemd drop-in
# Already configured on this machine at:
#   ~/.config/systemd/user/openclaw-gateway.service.d/op-env.conf
```

## Usage

```bash
# Source for shell access
source /home/openclaw/.openclaw/op-env.sh

# Add a found secret to vault
op item create --category="API Credential" --title="My API Key" \
  --vault "RhomBot's Vault" "credential=$SECRET_VALUE"

# Retrieve a secret
op item get "My API Key" --vault "RhomBot's Vault" --reveal --fields credential

# List vault contents
op item list --vault "RhomBot's Vault"
```

## Service Account vs Desktop App Integration

| | Service Account | Desktop App Integration |
|---|---|---|
| **WSL2** | ✅ Works | ❌ No PolKit D-Bus |
| **macOS/Linux desktop** | ✅ Works | ✅ Works |
| **Session duration** | Permanent token | 30-min session |
| **Audit logs** | ✅ Per service account | ✅ Per user |
| **Setup complexity** | Low | Medium |

## Security Notes

- Service account token is a secret — treat it like an API key
- It provides read/write access to the assigned vault
- Rotate it if compromised (same as any other API key)
- Access is logged in 1Password admin dashboard
- Never commit tokens to git — store in vault instead
