# ⚠️ DEPRECATED: Notion Encryption for Security Sweeps

> **This approach is deprecated as of 2026-03-30.** Use 1Password instead — see `1password.md` in this directory.

## Why 1Password is Better

- No blob pagination issues (Notion API truncates block content across pages)
- No master password to manage and risk losing
- Audit logs per access
- No encryption/decryption complexity
- Works reliably on WSL2 (no PolKit dependency)

## When Notion Encryption Was Used

- Before 2026-03-30: secrets were encrypted to Notion pages as AES-256-GCM blobs
- Notion-secrets.js was the encryption tool
- Master password stored in 1Password (the irony wasn't lost on us)

## If You Must Use It

The scripts and code are still in `~/.openclaw/scripts/notion-secrets.js` and the Notion secrets DB is archived but not deleted. If you need to recover old Notion-encrypted secrets:

1. Get the master password from 1Password ("Notion Master Password")
2. Fetch the archived pages from Notion
3. Decrypt with `node notion-secrets.js decrypt "<blob>"`
4. Migrate to 1Password as the new permanent home

## Migration Completed 2026-03-30

All 6 Notion-encrypted secrets (Gemini, Vercel, OpenAI, Anthropic, ClawHub, Resend) were decrypted and migrated to 1Password. The Notion secrets database pages were archived.

## Key Lesson

Notion API returns paginated block children. Always fetch full blocks:
```python
# Wrong — truncated blob
GET /blocks/{id}/children

# Right — iterate has_more
GET /blocks/{id}/children?page_size=100
while has_more:
    collect blocks
    GET /blocks/{id}/children?page_size=100&start_cursor={cursor}
```
This bit us — the first page appeared to work but the blob was incomplete.
