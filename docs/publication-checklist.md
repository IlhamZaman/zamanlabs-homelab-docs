# Public GitHub Publication Checklist

Before making this repository public:

- [ ] Search for private keys, API keys, tokens, passwords, cookies, and recovery codes.
- [ ] Confirm that `.env`, Terraform state, Ansible backups, databases, and application data are not present.
- [ ] Review screenshots separately because text-based secret scanning will not inspect pixels.
- [ ] Decide whether exact internal IP addresses, hostnames, domains, usernames, and filesystem paths should remain public.
- [ ] Confirm that example credentials are unmistakable placeholders.
- [ ] Verify that no commands accidentally expose a service directly to the internet.
- [ ] Check that old troubleshooting notes do not contain credentials copied from terminal output.
- [ ] Review `git diff --cached` before every push.
- [ ] Enable GitHub secret scanning and push protection when available.

## Recommended Initial Git Commands

```bash
cd zaman-labs-homelab-docs
git init
git add .
git status
git diff --cached
git commit -m "Add organized homelab documentation"
```

Create an empty GitHub repository, then add its remote and push only after the staged content has been reviewed.
