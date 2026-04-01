# Known Issues — ansible-core 2.17.x

## Breaking Changes from 2.16

### Remote Python 2.7 and Python 3.6 Support Removed
**Symptom:** Modules fail on managed nodes running Python 2.7 or Python 3.6.
**Cause:** Python 2.7 and Python 3.6 were dropped as supported remote Python versions. Python 3.7+ is now required on managed nodes.
**Workaround:** Upgrade managed nodes to Python 3.7 or newer.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### yum Module Removed, Redirected to dnf
**Symptom:** Playbooks referencing the `yum` module now execute `dnf` instead. The `yum` action plugin is also removed.
**Cause:** With the removal of Python 2 remote support, the `yum` module (which depended on Python 2 yum libraries) was removed and all calls are redirected to the `dnf` module.
**Workaround:** Switch playbooks to use `ansible.builtin.dnf` explicitly. Ensure `dnf` is available on target systems. For older RHEL/CentOS 7 systems that only have `yum`, use ansible-core 2.16 or earlier.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### Assert with Nested Templating May Fail
**Symptom:** `assert` conditions using nested Jinja2 templates may fail to evaluate.
**Cause:** Carried forward from 2.16.1 — templating tightening for conditionals.
**Workaround:** Simplify assert conditions to avoid nested template expressions.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### urls.py Python 2 Support Removed
**Symptom:** Custom modules using `ansible.module_utils.urls` may break on Python 2 environments.
**Cause:** Python 2 support was explicitly removed from `urls.py`.
**Workaround:** Ensure all managed nodes run Python 3.7+.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### scp_if_ssh Connection Option Removed
**Symptom:** Setting `scp_if_ssh` in the ssh connection plugin configuration produces an error.
**Cause:** The deprecated `scp_if_ssh` option was removed from the ssh connection plugin.
**Workaround:** Use the `ssh_transfer_method` option instead. Set `ssh_transfer_method = scp` if you need SCP behavior.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### JINJA2_NATIVE_WARNING Environment Variable Removed
**Symptom:** Setting `JINJA2_NATIVE_WARNING` environment variable has no effect.
**Cause:** The deprecated environment variable was removed.
**Workaround:** No action needed; the warning it controlled is no longer emitted.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

### Crypt Support Removed from ansible.utils.encrypt
**Symptom:** Password hashing using the `crypt` library via `ansible.utils.encrypt` fails.
**Cause:** Deprecated crypt support was removed.
**Workaround:** Use `passlib` for password hashing. Install it with `pip install passlib`.
**Affected versions:** 2.17.0 through latest
**Status:** Permanent — by design

## Security Advisories

### CVE-2024-0690 — ANSIBLE_NO_LOG Ignored
**Symptom:** Setting `ANSIBLE_NO_LOG` had no effect; sensitive data could appear in logs.
**Cause:** The `ANSIBLE_NO_LOG` environment variable was not being honored.
**Workaround:** Upgrade to ansible-core >= 2.17.0 which includes the fix from inception.
**Affected versions:** Pre-release versions only; fixed in 2.17.0
**Status:** Fixed in 2.17.0

### CVE-2023-5115 — Galaxy Role Symlink Path Traversal
**Symptom:** A malicious Galaxy role could use symlinks to overwrite files outside the installation directory.
**Cause:** Insufficient symlink validation during role installation.
**Workaround:** Included in 2.17.0 release. Audit roles before installing from untrusted sources.
**Affected versions:** Fixed in 2.17.0
**Status:** Fixed in 2.17.0

### CVE-2023-5764 — Unsafe Variables Lose Designation via Templating
**Symptom:** Internal templating can cause variables marked as `AnsibleUnsafe` to lose their unsafe designation.
**Cause:** A flaw in internal template processing.
**Workaround:** Included in 2.17.0 release.
**Affected versions:** Fixed in 2.17.0
**Status:** Fixed in 2.17.0

### CVE-2024-8775 — Vault-Encrypted Variables Exposed via include_vars
**Symptom:** Vault-encrypted file contents could be exposed in task results when loaded via `include_vars`.
**Cause:** Result masking was not correctly requested when vault-encrypted files were read.
**Workaround:** Upgrade to ansible-core >= 2.17.6.
**Affected versions:** 2.17.0 through 2.17.5
**Status:** Fixed in 2.17.6

### CVE-2024-9902 — User Module SSH Key Symlink Traversal
**Symptom:** The `user` module could follow symlinks when running `ssh-keygen`, `chown`, and `chmod` on existing SSH public key files.
**Cause:** Insufficient symlink validation on existing SSH public key files.
**Workaround:** Upgrade to ansible-core >= 2.17.6.
**Affected versions:** 2.17.0 through 2.17.5
**Status:** Fixed in 2.17.6

### CVE-2024-11079 — AnsibleUnsafe Bypass via hostvars
**Symptom:** Referencing a variable via `hostvars` could cause the `AnsibleUnsafe` designation to be lost, potentially allowing template injection.
**Cause:** A flaw in how hostvars references interact with the unsafe designation system.
**Workaround:** Upgrade to ansible-core >= 2.17.7.
**Affected versions:** 2.17.0 through 2.17.6
**Status:** Fixed in 2.17.7

## Known Regressions

### Rapid Memory Growth with Handler `listen` Keyword
**Symptom:** Memory usage grows rapidly when notifying handlers using the `listen` keyword, potentially causing OOM on large playbook runs.
**Cause:** A memory leak in handler notification tracking.
**Workaround:** Upgrade to ansible-core >= 2.17.1.
**Affected versions:** 2.17.0
**Status:** Fixed in 2.17.1

### Environment Variable with Special Characters Causes Traceback
**Symptom:** A traceback occurs when an environment variable passed to a task contains certain special characters.
**Cause:** Improper handling of special characters in environment variable processing.
**Workaround:** Upgrade to ansible-core >= 2.17.2. As a temporary workaround, avoid special characters in environment variables or quote them carefully.
**Affected versions:** 2.17.0 through 2.17.1
**Status:** Fixed in 2.17.2

### Handler Execution Not in Lockstep with Linear Strategy
**Symptom:** Handlers included via `include_tasks` are not executed in lockstep across hosts when using the linear strategy.
**Cause:** A regression in handler scheduling within the linear strategy.
**Workaround:** Upgrade to ansible-core >= 2.17.3.
**Affected versions:** 2.17.0 through 2.17.2
**Status:** Fixed in 2.17.3

### Shell Plugin Quoting Regression
**Symptom:** Shell commands with special characters fail or behave unexpectedly.
**Cause:** Shell plugin did not properly quote all needed components of shell commands.
**Workaround:** Upgrade to ansible-core >= 2.17.1.
**Affected versions:** 2.17.0
**Status:** Fixed in 2.17.1

### dnf5 Missing `state: installed` Alias
**Symptom:** Using `state: installed` with the `dnf5` module fails; only `state: present` works.
**Cause:** The `installed` alias was unintentionally removed in the dnf5 module.
**Workaround:** Use `state: present` instead, or upgrade to ansible-core >= 2.17.5.
**Affected versions:** 2.17.0 through 2.17.4
**Status:** Fixed in 2.17.5

### gather_facts with `smart` Setting Fails to Default to setup
**Symptom:** The `gather_facts` action does not fall back to `ansible.legacy.setup` when `smart` is configured but no network OS is detected.
**Cause:** Missing default fallback logic in the gather_facts action.
**Workaround:** Explicitly set `ansible.builtin.setup` in `FACTS_MODULES`, or upgrade to ansible-core >= 2.17.8.
**Affected versions:** 2.17.0 through 2.17.7
**Status:** Fixed in 2.17.8

## Completed Deprecations (Features Removed in 2.17)

| Removed Feature | Replacement |
|---|---|
| `yum` module and action plugin | `ansible.builtin.dnf` |
| Python 2.7 and 3.6 remote support | Python 3.7+ on managed nodes |
| `scp_if_ssh` ssh connection option | `ssh_transfer_method` option |
| `JINJA2_NATIVE_WARNING` env variable | No longer needed |
| `crypt` support in `ansible.utils.encrypt` | Use `passlib` |
| `ansible-docs` deprecated APIs | Use current API |
| `virtualenv` fallback in ansible-test | Use `python -m venv` |
| Various Python 2-era sanity tests (`no-basestring`, `no-dict-iteritems`, `no-dict-iterkeys`, `no-dict-itervalues`, `no-unicode-literals`) | No longer needed with Python 3 |

## Deprecations Announced in 2.17 (Removals Planned for Future Versions)

| Deprecated Feature | Replacement | Target Removal |
|---|---|---|
| Old style vars plugins (`get_host_vars`/`get_group_vars`) | Inherit from `BaseVarsPlugin`, define `get_vars` | 2.20+ |
| `get_bin_path` `required` parameter | Handle missing binary in module logic | 2.20+ |
| Various `ansible.module_utils.basic` convenience imports (`get_exception`, `literal_eval`, `PY2`, `PY3`, etc.) | Import from standard library directly | 2.20+ |
| Galaxy v2 API usage | Use Galaxy v3 API | Future |
| Paramiko global scope config items | Use plugin-level `get_option()` | 2.20+ |
| Role entrypoint attributes in ansible-doc | Will be removed from output | 2.20 |

## Python Version Requirements

| Component | Minimum Python | Notes |
|---|---|---|
| Controller | 3.10 | Same as 2.16 |
| Managed node | 3.7 | Raised from 2.7/3.6 in 2.16 |
| setuptools | 66.1.0 | Required for Python 3.12 support |
