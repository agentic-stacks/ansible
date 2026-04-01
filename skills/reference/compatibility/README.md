# Compatibility Matrices

Version compatibility reference for ansible-core, Python, OS platforms, and collections.

Source: https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html

---

## Python Version Requirements

| ansible-core | Control Node Python | Managed Node Python        |
|--------------|---------------------|----------------------------|
| 2.20         | 3.12 -- 3.14        | 3.9 -- 3.14                |
| 2.19         | 3.11 -- 3.13        | 3.8 -- 3.13                |
| 2.18         | 3.11 -- 3.13        | 3.8 -- 3.13                |
| 2.17         | 3.10 -- 3.12        | 3.7 -- 3.12                |
| 2.16 (EOL)   | 3.10 -- 3.12        | 2.7, 3.6 -- 3.12           |
| 2.15 (EOL)   | 3.9 -- 3.11         | 2.7, 3.5 -- 3.11           |

Key changes across versions:

- **2.17** dropped Python 2.7 managed node support. Minimum managed node is now Python 3.7.
- **2.18** raised managed node minimum to Python 3.8 and control node minimum to Python 3.11.
- **2.19** kept the same Python ranges as 2.18.
- **2.20** raised control node minimum to Python 3.12, managed node minimum to Python 3.9.

Windows managed nodes use **PowerShell 5.1** (all current versions).

---

## OS Support

| OS               | Control Node       | Managed Node       |
|------------------|--------------------|---------------------|
| Ubuntu 22.04+    | Yes                | Yes                 |
| RHEL / CentOS 8+ | Yes                | Yes                 |
| Debian 11+       | Yes                | Yes                 |
| macOS 13+        | Yes (pip only)     | No                  |
| Windows          | No                 | Yes (WinRM / PSRP)  |
| FreeBSD          | Community          | Yes                 |

Notes:

- Control node requires a POSIX system (Linux, macOS, BSDs). Windows is **not** supported as a control node.
- macOS control node installs must use `pip`; the Homebrew `ansible` formula may lag behind.
- Windows managed nodes require WinRM or PSRP (PowerShell Remoting Protocol) and `pywinrm` on the control node.

---

## Ansible Community Package to ansible-core Mapping

| Ansible Package | ansible-core | Status                      |
|-----------------|--------------|-----------------------------|
| 14.x            | 2.21         | In development (unreleased) |
| 13.x            | 2.20         | Current (latest)            |
| 12.x            | 2.19         | EOL Dec 2025                |
| 11.x            | 2.18         | EOL Dec 2025                |
| 10.x            | 2.17         | End of life                 |
| 9.x             | 2.16         | End of life                 |
| 8.x             | 2.15         | End of life                 |
| 7.x             | 2.14         | End of life                 |

The community `ansible` package bundles curated collections on top of `ansible-core`. Install the package version that matches your required `ansible-core`:

```bash
# Install specific major version
pip install 'ansible>=13,<14'   # gets ansible-core 2.20

# Or install ansible-core directly (no bundled collections)
pip install 'ansible-core>=2.20,<2.21'
```

---

## Collection Compatibility

Collections declare their supported `ansible-core` versions in `meta/runtime.yml`:

```yaml
# Example: meta/runtime.yml inside a collection
requires_ansible: '>=2.15.0'
```

### Checking installed collection versions

```bash
ansible-galaxy collection list
```

### Pinning collections in requirements.yml

```yaml
# requirements.yml
collections:
  - name: community.general
    version: '>=8.0.0,<9.0.0'
  - name: ansible.posix
    version: '>=1.5.0'
  - name: community.aws
    version: '>=7.0.0'
```

Install pinned collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

### Verifying compatibility

If a collection requires a newer `ansible-core` than what is installed, `ansible-galaxy` will raise an error at install time. To check proactively:

```bash
# Show collection metadata including ansible-core requirement
ansible-galaxy collection info community.general
```

### Best practices

- Pin collection versions in `requirements.yml` for reproducible environments.
- When upgrading `ansible-core`, check that all collections still list your version as supported.
- Prefer version ranges (`>=8.0.0,<9.0.0`) over exact pins to receive patch updates.
- Run `ansible-galaxy collection verify` to detect modified or corrupted collections.
