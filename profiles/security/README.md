# Security Profile

## Overview
Hardened Ansible configuration for security-sensitive environments.

## Settings
- Vault-first: all secrets encrypted with ansible-vault, no exceptions
- Host key checking: always enabled (ANSIBLE_HOST_KEY_CHECKING=True)
- Become method: restricted to `sudo` (no `su`, `pbrun`, etc.)
- No `command` or `shell` modules without `creates`/`removes` guard
- ansible-lint profile: `safety` minimum

## ansible.cfg Overrides

```ini
[defaults]
host_key_checking = True
retry_files_enabled = False

[privilege_escalation]
become_method = sudo
```

## Applicable Skills
- `skills/operations/vault` — mandatory for all variable files containing credentials
- `skills/operations/testing` — ansible-lint `safety` profile required in CI
