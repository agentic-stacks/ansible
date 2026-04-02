# Molecule

Molecule tests Ansible roles by converging them inside real instances (containers or VMs) and verifying the result. It catches convergence failures, idempotence issues, and broken role logic before code reaches production.

## Install

Install Molecule and the Docker driver plugin:

```bash
pip install molecule molecule-plugins[docker]
```

The `molecule-plugins` package provides drivers. Install the extras you need: `[docker]`, `[podman]`, or `[vagrant]`.

```bash
# Verify installation
molecule --version
```

## Initialize a Scenario

Run `molecule init scenario` inside an existing role directory to scaffold the default test scenario:

```bash
cd roles/my_role
molecule init scenario -d docker
```

This creates a `molecule/default/` directory with:

```
roles/my_role/
  molecule/
    default/
      molecule.yml        # scenario configuration
      converge.yml        # playbook that applies the role
      verify.yml          # playbook that tests the result
      create.yml          # (optional) custom instance creation
      destroy.yml         # (optional) custom instance teardown
      prepare.yml         # (optional) pre-convergence setup
```

## Scenario Lifecycle

`molecule test` runs the full lifecycle in this exact order:

1. **dependency** — Install role/collection dependencies from `requirements.yml`
2. **lint** — Run linters (deprecated in recent versions; use ansible-lint separately)
3. **cleanup** — Run cleanup playbook (pre-destroy housekeeping)
4. **destroy** — Tear down any existing instances from a prior run
5. **syntax** — `ansible-playbook --syntax-check` on converge.yml
6. **create** — Spin up test instances
7. **prepare** — Run prepare.yml (install prerequisites on instances)
8. **converge** — Apply the role via converge.yml
9. **idempotence** — Converge again, fail if any task reports "changed"
10. **side_effect** — Run optional side-effect playbook (simulate failures, etc.)
11. **verify** — Run verify.yml assertions
12. **cleanup** — Run cleanup playbook (post-test housekeeping)
13. **destroy** — Tear down instances

## molecule.yml Configuration

The scenario configuration file defines the driver, platforms, provisioner, and verifier:

```yaml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: docker

platforms:
  - name: ubuntu-test
    image: ubuntu:22.04
    pre_build_image: true
    command: /bin/systemd
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
  - name: rocky-test
    image: rockylinux:9
    pre_build_image: true
    command: /bin/systemd
    privileged: true

provisioner:
  name: ansible
  config_options:
    defaults:
      callbacks_enabled: profile_tasks
  inventory:
    host_vars:
      ubuntu-test:
        my_var: ubuntu_value
      rocky-test:
        my_var: rocky_value

verifier:
  name: ansible
```

Key sections:

- **driver** — Which backend creates instances (docker, podman, delegated).
- **platforms** — One or more target instances. `pre_build_image: true` skips Dockerfile generation and pulls the image directly.
- **provisioner** — Ansible configuration. Pass extra variables, inventory, or `ansible.cfg` options here.
- **verifier** — How to verify. Default is `ansible` (runs verify.yml). Alternative: `testinfra`.

## Write Tests (Verifier)

### Default Ansible Verifier

Write assertions in `molecule/default/verify.yml`:

```yaml
---
- name: Verify
  hosts: all
  gather_facts: true
  tasks:
    - name: Check that nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present
      check_mode: true
      register: pkg_result
      failed_when: pkg_result.changed

    - name: Check that nginx service is running
      ansible.builtin.service:
        name: nginx
        state: started
      check_mode: true
      register: svc_result
      failed_when: svc_result.changed

    - name: Check that nginx config is valid
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Check that port 80 is listening
      ansible.builtin.wait_for:
        port: 80
        timeout: 5

    - name: Check config file content
      ansible.builtin.slurp:
        src: /etc/nginx/nginx.conf
      register: nginx_conf

    - name: Assert worker_processes is set
      ansible.builtin.assert:
        that:
          - "'worker_processes' in nginx_conf.content | b64decode"
        fail_msg: "worker_processes directive missing from nginx.conf"
```

### Testinfra Alternative

If you prefer Python-based tests, switch the verifier and write pytest-style tests:

```yaml
# molecule.yml
verifier:
  name: testinfra
```

```python
# molecule/default/tests/test_default.py
import pytest

def test_nginx_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed

def test_nginx_running(host):
    nginx = host.service("nginx")
    assert nginx.is_running
    assert nginx.is_enabled

def test_nginx_listening(host):
    assert host.socket("tcp://0.0.0.0:80").is_listening
```

## Run Tests

```bash
# Full lifecycle (create, converge, verify, destroy)
molecule test

# Apply the role only (skip verify and destroy) — fast iteration
molecule converge

# Run verification tests against already-converged instances
molecule verify

# SSH into a running instance for debugging
molecule login --host ubuntu-test

# Destroy instances manually
molecule destroy

# List instances and their status
molecule list
```

Use `molecule converge` and `molecule verify` during development for fast feedback. Run `molecule test` in CI for the full lifecycle.

## Drivers

| Driver | Speed | Use Case | Notes |
|--------|-------|----------|-------|
| docker | Fast | Default choice, CI pipelines | Requires Docker daemon. Lightweight containers. |
| podman | Fast | Rootless containers, RHEL environments | No daemon needed. Drop-in replacement for Docker. |
| delegated | Varies | Custom infrastructure (cloud VMs, bare metal) | You write create.yml and destroy.yml yourself. |

Install the driver you need:

```bash
# Docker (most common)
pip install molecule-plugins[docker]

# Podman
pip install molecule-plugins[podman]

# Delegated requires no extra install — you provide create/destroy playbooks
```

## CI Integration

GitHub Actions workflow example:

```yaml
# .github/workflows/molecule.yml
---
name: Molecule Test
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  molecule:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        role:
          - my_role
          - another_role
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install ansible molecule molecule-plugins[docker]

      - name: Run Molecule
        working-directory: roles/${{ matrix.role }}
        run: molecule test
        env:
          PY_COLORS: "1"
          ANSIBLE_FORCE_COLOR: "1"
```

Set `PY_COLORS` and `ANSIBLE_FORCE_COLOR` to get colored output in CI logs.

## References

- [Molecule documentation](https://ansible.readthedocs.io/projects/molecule/)
- [Molecule configuration reference](https://ansible.readthedocs.io/projects/molecule/configuration/)
- [molecule-plugins (drivers)](https://github.com/ansible/molecule-plugins)
