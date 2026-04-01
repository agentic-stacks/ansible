# Install Ansible

This skill covers every supported method for installing Ansible on a control node, verifying the installation, managing collections, and pinning versions for reproducible automation.

## Meet the Prerequisites

### Python version requirements by ansible-core version

| `ansible-core` | Ansible package | Control node Python | Managed node Python |
|-----------------|-----------------|---------------------|---------------------|
| 2.18 | 11.x | 3.11 -- 3.13 | 3.8 -- 3.13 |
| 2.17 | 10.x | 3.10 -- 3.12 | 3.7 -- 3.12 |
| 2.16 | 9.x | 3.10 -- 3.12 | 2.7, 3.6 -- 3.12 |
| 2.15 | 8.x | 3.9 -- 3.11 | 2.7, 3.5 -- 3.11 |

<!-- version-marker: ansible-core 2.19 (GA July 2025) requires Python 3.11+ on the control node. -->

> **Managed nodes** do not need Ansible installed. They need Python (see table above) and an SSH-reachable user account with an interactive POSIX shell. Network modules are an exception — they do not require Python on the managed device.

Verify your Python version before proceeding:

```bash
python3 --version
```

## Install via pip (Recommended)

Use a virtual environment to isolate Ansible and its dependencies from system Python packages.

### Create and activate a virtual environment

```bash
python3 -m venv ~/.venv/ansible
source ~/.venv/ansible/bin/activate
```

### Install the full community package (core + collections)

```bash
pip install ansible
```

This installs `ansible-core` plus ~85 curated community collections from Ansible Galaxy.

### Install ansible-core only (minimal)

```bash
pip install ansible-core
```

### Pin to a specific version

```bash
pip install ansible-core==2.17.0
pip install ansible==10.0.1
```

### Install with pipx (alternative)

On systems where `pip install` is restricted (e.g., externally-managed Python environments), use `pipx`:

```bash
pipx install --include-deps ansible
pipx install ansible-core==2.17.0
```

To inject extra Python dependencies when using pipx:

```bash
pipx inject ansible argcomplete
pipx inject --include-apps ansible argcomplete
```

## Install via OS Package Manager

OS-packaged versions may lag behind the latest pip release. Check the version after installing.

### Ubuntu / Debian (apt)

```bash
sudo apt update
sudo apt install ansible
```

### RHEL / Fedora (dnf)

```bash
sudo dnf install ansible
```

### macOS (Homebrew)

```bash
brew install ansible
```

> **Note:** OS packages often bundle an older `ansible-core` version. For the latest release and full control over versions, prefer the pip method above.

## Verify the Installation

Run these commands to confirm a working installation:

```bash
# Show ansible-core version and configuration paths
ansible --version

# Show the community package version (if installed via pip install ansible)
ansible-community --version

# Confirm the Python interpreter in use
python3 --version

# List installed collections
ansible-galaxy collection list
```

Expected output from `ansible --version` includes the core version, configuration file path, configured module search path, Python interpreter location, and jinja2/libyaml versions.

## Install Specific Collections

### Install a single collection

```bash
ansible-galaxy collection install community.general
```

### Install a pinned version

```bash
ansible-galaxy collection install community.general:==9.5.0
```

### Install from a requirements file

Create `requirements.yml` to declare all required collections with pinned versions:

```yaml
---
collections:
  - name: community.general
    version: ">=9.0.0,<10.0.0"
  - name: amazon.aws
    version: "==8.2.1"
  - name: ansible.posix
    version: ">=1.5.0"
```

Install everything in the file:

```bash
ansible-galaxy collection install -r requirements.yml
```

To force reinstall (e.g., after changing pinned versions):

```bash
ansible-galaxy collection install -r requirements.yml --force
```

## Pin Versions (Best Practices)

Pinning Ansible and collection versions prevents unexpected breakage when upgrading.

### Freeze current pip packages

```bash
pip freeze > requirements.txt
```

### Use a constraints file

Create `constraints.txt` to set upper bounds without locking every transitive dependency:

```
ansible-core>=2.17.0,<2.18.0
jinja2>=3.1.0,<3.2.0
```

Install with constraints:

```bash
pip install ansible-core -c constraints.txt
```

### Recommended requirements.txt pattern

```
ansible-core==2.17.6
jinja2==3.1.4
PyYAML==6.0.2
```

Combine with collection pinning in `requirements.yml` (see above) for full reproducibility.

## Upgrade Ansible

> **Warning:** Always check `skills/reference/known-issues` before upgrading.

### Upgrade ansible-core via pip

```bash
pip install --upgrade ansible-core
```

### Upgrade the full community package

```bash
pip install --upgrade ansible
```

### Upgrade via pipx

```bash
pipx upgrade --include-injected ansible
```

### Upgrade a single collection

```bash
ansible-galaxy collection install community.general --upgrade
```

After upgrading, verify the new version:

```bash
ansible --version
ansible-galaxy collection list
```

---

## References

These official documentation pages were used to verify the content in this skill:

- [Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Installation Guide (index)](https://docs.ansible.com/ansible/latest/installation_guide/index.html)
- [Release and maintenance — ansible-core support matrix](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html)
- [Installing Ansible on specific operating systems](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)
