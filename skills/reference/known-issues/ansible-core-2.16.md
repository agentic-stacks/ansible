# Known Issues — ansible-core 2.16.x

## Breaking Changes from 2.15

### Controller Python 3.9 Support Removed
**Symptom:** ansible-core 2.16 fails to run on Python 3.9 on the controller node.
**Cause:** Python 3.9 was removed as a supported controller version. Python 3.10 or newer is now required.
**Workaround:** Upgrade the controller node to Python 3.10+.
**Affected versions:** 2.16.0 through latest
**Status:** Permanent — by design

### Remote Python 3.5 Support Removed
**Symptom:** Modules fail on remote hosts running Python 3.5.
**Cause:** Python 3.5 was dropped as a supported remote Python version. Python 2.7 or Python 3.6+ is now required on managed nodes.
**Workaround:** Upgrade managed nodes to Python 3.6+ or use Python 2.7.
**Affected versions:** 2.16.0 through latest
**Status:** Permanent — by design

### The `include` Module Is Removed (Tombstoned)
**Symptom:** Playbooks using the bare `include` directive fail with a tombstone error.
**Cause:** `include` was deprecated in Ansible 2.12 and removed in 2.16.
**Workaround:** Replace `include` with `include_tasks` or `import_tasks`.
**Affected versions:** 2.16.0 through latest
**Status:** Permanent — by design

### Default Transport Changed from `smart` to `ssh`
**Symptom:** Connection behavior changes for users who relied on the `smart` transport automatically selecting `paramiko` when OpenSSH ControlPersist was unavailable.
**Cause:** `DEFAULT_TRANSPORT` now defaults to `ssh`. The `smart` option is deprecated.
**Workaround:** Explicitly set `transport = paramiko` in `ansible.cfg` if you require paramiko.
**Affected versions:** 2.16.0 through latest
**Status:** Permanent — by design

### Service Module OpenBSD Behavior Change
**Symptom:** The `service` module on OpenBSD no longer permanently configures variables/flags during enable/disable operations.
**Cause:** The module was changed to only manage service state, not persistent configuration.
**Workaround:** Use `rcctl` directly or create a dedicated `rcctl_config` task for persistent variable configuration.
**Affected versions:** 2.16.0 through latest
**Status:** Permanent — by design

### Assert with Nested Templating May Fail
**Symptom:** `assert` conditions using nested Jinja2 templates may fail to evaluate, returning errors instead of the expected boolean result.
**Cause:** Templating changes in 2.16.1 tightened how conditionals are evaluated.
**Workaround:** Simplify assert conditions to avoid nested template expressions. Ensure conditions resolve to a strict boolean.
**Affected versions:** 2.16.1 through latest
**Status:** Permanent — by design

## Security Advisories

### CVE-2023-5764 — Unsafe Variables Lose Designation via Templating
**Symptom:** Internal templating can cause variables marked as `AnsibleUnsafe` to lose their unsafe designation, potentially allowing template injection.
**Cause:** A flaw in internal template processing.
**Workaround:** Upgrade to ansible-core >= 2.16.1 which contains the fix.
**Affected versions:** 2.16.0
**Status:** Fixed in 2.16.1

### CVE-2023-5115 — Galaxy Role Symlink Path Traversal
**Symptom:** A malicious Galaxy role could use symlinks to overwrite files outside the installation directory.
**Cause:** Insufficient symlink validation during role installation.
**Workaround:** Upgrade to ansible-core >= 2.16.0 which contains the fix. Audit roles before installing from untrusted sources.
**Affected versions:** Versions prior to 2.16.0 (fix included in 2.16.0 release)
**Status:** Fixed in 2.16.0

### CVE-2024-0690 — ANSIBLE_NO_LOG Ignored
**Symptom:** Setting `ANSIBLE_NO_LOG` had no effect; sensitive data could appear in logs.
**Cause:** The `ANSIBLE_NO_LOG` environment variable was not being honored.
**Workaround:** Upgrade to ansible-core >= 2.16.3.
**Affected versions:** 2.16.0 through 2.16.2
**Status:** Fixed in 2.16.3

### CVE-2024-8775 — Vault-Encrypted Variables Exposed via include_vars
**Symptom:** Vault-encrypted file contents could be exposed in task results when loaded via `include_vars`.
**Cause:** Result masking was not correctly requested when vault-encrypted files were read.
**Workaround:** Upgrade to ansible-core >= 2.16.13.
**Affected versions:** 2.16.0 through 2.16.12
**Status:** Fixed in 2.16.13

### CVE-2024-9902 — User Module SSH Key Symlink Traversal
**Symptom:** The `user` module could follow symlinks when running `ssh-keygen`, `chown`, and `chmod` on existing SSH public key files, allowing potential file traversal.
**Cause:** Insufficient symlink validation on existing SSH public key files.
**Workaround:** Upgrade to ansible-core >= 2.16.13.
**Affected versions:** 2.16.0 through 2.16.12
**Status:** Fixed in 2.16.13

### CVE-2024-11079 — AnsibleUnsafe Bypass via hostvars
**Symptom:** Referencing a variable via `hostvars` could cause the templating engine to prefer `AnsibleUnsafe` designation to be lost, potentially allowing template injection.
**Cause:** A flaw in how hostvars references interact with the unsafe designation system.
**Workaround:** Upgrade to ansible-core >= 2.16.14.
**Affected versions:** 2.16.0 through 2.16.13
**Status:** Fixed in 2.16.14

## Known Regressions

### Role Params Precedence Regression
**Symptom:** Role parameters have lower precedence than host facts, contradicting documentation.
**Cause:** An unintentional change in 2.15 altered variable precedence. The fix was backported.
**Workaround:** Upgrade to ansible-core >= 2.16.1 where role params once again have higher precedence than host facts.
**Affected versions:** 2.16.0
**Status:** Fixed in 2.16.1

### Handler Execution Not in Lockstep with Linear Strategy
**Symptom:** Handlers are not executed in lockstep across hosts when using the linear strategy.
**Cause:** A regression in handler scheduling with the linear strategy.
**Workaround:** Upgrade to ansible-core >= 2.16.7.
**Affected versions:** 2.16.0 through 2.16.6
**Status:** Fixed in 2.16.7

### run_command Output Truncation or Hang
**Symptom:** Module output is truncated or tasks hang indefinitely when using `run_command`.
**Cause:** Premature selector unregistration on empty read from stdout/stderr.
**Workaround:** Upgrade to ansible-core >= 2.16.15.
**Affected versions:** 2.16.0 through 2.16.14
**Status:** Fixed in 2.16.15

### dnf5 Missing `state: installed` Alias
**Symptom:** Using `state: installed` with the `dnf5` module fails; only `state: present` works.
**Cause:** The `installed` alias was unintentionally removed in the dnf5 module.
**Workaround:** Use `state: present` instead, or upgrade to ansible-core >= 2.16.12.
**Affected versions:** 2.16.0 through 2.16.11
**Status:** Fixed in 2.16.12

## Completed Deprecations (Features Removed in 2.16)

| Removed Feature | Replacement |
|---|---|
| `include` task directive | `include_tasks` or `import_tasks` |
| `ActionBase._remote_checksum` method | Use the updated API |
| `PlayIterator.cache_block_tasks` / `get_original_task` | Removed without replacement |
| `FileLock` class | Use standard library alternatives |
| `Templar(shared_loader_obj=...)` parameter | Remove the parameter from calls |
| `fetch_url` auto-disable `decompress` | Ensure gzip is available |
| `stat` module `get_md5` parameter | Use `checksum_algorithm` instead |
| `default.fact_caching_prefix` ini option | Use `defaults.fact_caching_prefix` |
| Python 3.9 controller support | Upgrade to Python 3.10+ |
| Python 3.5 remote support | Use Python 2.7 or 3.6+ |
| Windows Server 2012/2012-R2 targets | Use Windows Server 2016+ |

## Python Version Requirements

| Component | Minimum Python | Notes |
|---|---|---|
| Controller | 3.10 | Raised from 3.9 in 2.15 |
| Managed node | 2.7 or 3.6 | Python 3.5 removed |
| setuptools | 66.1.0 | Required for Python 3.12 support |
