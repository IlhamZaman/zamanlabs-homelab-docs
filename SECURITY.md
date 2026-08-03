# Security and Publication Policy

This repository may be published, but it must never contain live credentials or files that can be used to access the homelab.

## Never Commit

- SSH private keys
- Cloudflare API tokens or tunnel credentials
- Discord bot tokens
- OpenAI API keys or OAuth credentials
- Ansible Vault passwords
- `.env` files
- Terraform state or plan files
- TrueNAS, Proxmox, database, or application passwords
- Session cookies, bearer tokens, private certificates, or recovery codes
- Backups containing `/etc/shadow`, private keys, application databases, or secrets

## Before Every Push

Run a secret scan and inspect the staged diff:

```bash
git diff --cached

grep -RniE '(BEGIN (RSA|OPENSSH|EC) PRIVATE KEY|sk-[A-Za-z0-9_-]{20,}|api[_-]?key|token|password|secret)' . \
  --exclude-dir=.git \
  --exclude='SECURITY.md'
```

A matching word is not automatically a leaked secret, but every match must be reviewed.

## Reporting a Problem

Do not open a public issue containing a credential or sensitive screenshot. Revoke the credential first, remove it from Git history, and then document the incident without the secret value.
