# ansible-test

`ansible-test` is the official testing tool for Ansible collections. It runs sanity checks, unit tests, and integration tests against collection code.

## When to Use

Use `ansible-test` when developing **collections only**. It expects the standard collection directory layout and must be run from inside the collection root.

Do **not** use `ansible-test` for:

- Standalone playbooks — use `ansible-lint` instead
- Standalone roles — use `Molecule` instead
- Anything outside a `collections/ansible_collections/<namespace>/<name>/` path

`ansible-test` ships with `ansible-core`. No separate install is needed:

```bash
# Verify it is available
ansible-test --version
```

## Collection Directory Layout

`ansible-test` requires this structure:

```
collections/ansible_collections/my_namespace/my_collection/
  galaxy.yml
  plugins/
    modules/
      my_module.py
    module_utils/
      my_util.py
  roles/
  tests/
    unit/
      plugins/
        modules/
          test_my_module.py
    integration/
      targets/
        my_module/
          tasks/
            main.yml
    sanity/
      ignore-2.17.txt
```

## Sanity Tests

Sanity tests validate code quality without running the modules. They check for import errors, documentation completeness, PEP 8 compliance, GPL headers, and more.

```bash
# Run all sanity tests
ansible-test sanity

# Run against a specific Python version
ansible-test sanity --python 3.12

# Run a specific sanity test
ansible-test sanity --test pep8
ansible-test sanity --test import
ansible-test sanity --test validate-modules

# List available sanity tests
ansible-test sanity --list-tests
```

Common sanity tests:

| Test | What It Checks |
|------|---------------|
| `pep8` | Python code style (PEP 8 compliance) |
| `import` | Modules import without errors |
| `validate-modules` | Module documentation, argument spec, return docs |
| `pylint` | Static analysis of Python code |
| `compile` | Python files compile without syntax errors |
| `yamllint` | YAML file syntax and formatting |
| `no-unwanted-files` | No unexpected files in the collection |

Skip known failures with an ignore file at `tests/sanity/ignore-<version>.txt`:

```
plugins/modules/my_module.py pep8 E501
plugins/modules/legacy.py import --allow-disabled
```

## Unit Tests

Unit tests verify individual functions and classes in isolation using `pytest`.

### Directory Structure

```
tests/
  unit/
    plugins/
      modules/
        test_my_module.py
      module_utils/
        test_my_util.py
```

The path under `tests/unit/` must mirror the path under `plugins/`.

### Writing a Unit Test

```python
# tests/unit/plugins/modules/test_my_module.py
from __future__ import absolute_import, division, print_function
__metaclass__ = type

import pytest
from unittest.mock import patch, MagicMock
from ansible_collections.my_namespace.my_collection.plugins.modules import my_module


class TestMyModule:
    def test_argument_spec(self):
        """Verify the module argument spec is valid."""
        spec = my_module.argument_spec()
        assert "name" in spec
        assert spec["name"]["type"] == "str"
        assert spec["name"]["required"] is True

    @patch("ansible_collections.my_namespace.my_collection.plugins.modules.my_module.AnsibleModule")
    def test_create_resource(self, mock_module):
        mock_module.return_value.params = {
            "name": "test",
            "state": "present",
        }
        mock_module.return_value.check_mode = False
        my_module.main()
        mock_module.return_value.exit_json.assert_called_once()
```

### Running Unit Tests

```bash
# Run all unit tests
ansible-test units

# Run against a specific Python version
ansible-test units --python 3.12

# Run a specific test file
ansible-test units tests/unit/plugins/modules/test_my_module.py

# Run with verbose output
ansible-test units -v
```

## Integration Tests

Integration tests run modules against real or simulated infrastructure. Each test is a "target" — a directory containing an Ansible tasks file.

### Directory Structure

```
tests/
  integration/
    targets/
      my_module/
        tasks/
          main.yml
        defaults/
          main.yml
        meta/
          main.yml
      my_module_cleanup/
        tasks/
          main.yml
```

### Writing an Integration Test

```yaml
# tests/integration/targets/my_module/tasks/main.yml
---
- name: Create a resource
  my_namespace.my_collection.my_module:
    name: test-resource
    state: present
  register: create_result

- name: Assert resource was created
  ansible.builtin.assert:
    that:
      - create_result.changed
      - create_result.resource.name == "test-resource"

- name: Create the same resource again (idempotence)
  my_namespace.my_collection.my_module:
    name: test-resource
    state: present
  register: idem_result

- name: Assert no change on second run
  ansible.builtin.assert:
    that:
      - not idem_result.changed

- name: Delete the resource
  my_namespace.my_collection.my_module:
    name: test-resource
    state: absent
  register: delete_result

- name: Assert resource was deleted
  ansible.builtin.assert:
    that:
      - delete_result.changed
```

### Running Integration Tests

```bash
# Run all integration tests
ansible-test integration

# Run a specific target
ansible-test integration my_module

# Run with verbose output
ansible-test integration -v my_module

# Continue on failure
ansible-test integration --continue-on-error
```

## Running in Docker

Docker isolates tests from the host and ensures a clean environment. Pass `--docker` to any test command:

```bash
# Sanity tests in Docker
ansible-test sanity --docker default

# Unit tests in Docker
ansible-test units --docker default

# Integration tests in Docker
ansible-test integration --docker default my_module

# Use a specific Docker image
ansible-test sanity --docker ubuntu2204
ansible-test sanity --docker default --python 3.12
```

Available Docker images include `default`, `ubuntu2204`, `alpine`, and others. The `default` image is recommended for most cases.

Docker mode is strongly recommended for CI pipelines because it guarantees consistent results regardless of the host environment.

## CI Configuration

GitHub Actions example for testing a collection:

```yaml
# .github/workflows/collection-test.yml
---
name: Collection Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sanity:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          path: ansible_collections/my_namespace/my_collection

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install ansible-core
        run: pip install ansible-core

      - name: Run sanity tests
        working-directory: ansible_collections/my_namespace/my_collection
        run: ansible-test sanity --docker default --python ${{ matrix.python-version }}

  units:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          path: ansible_collections/my_namespace/my_collection

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install ansible-core
        run: pip install ansible-core

      - name: Run unit tests
        working-directory: ansible_collections/my_namespace/my_collection
        run: ansible-test units --docker default

  integration:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          path: ansible_collections/my_namespace/my_collection

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install ansible-core
        run: pip install ansible-core

      - name: Run integration tests
        working-directory: ansible_collections/my_namespace/my_collection
        run: ansible-test integration --docker default
```

The `path:` field in the checkout step is critical. `ansible-test` must find itself inside a `ansible_collections/<namespace>/<name>/` directory structure or it will refuse to run.

## References

- [Testing Ansible collections](https://docs.ansible.com/ansible/latest/dev_guide/testing.html)
- [ansible-test sanity tests](https://docs.ansible.com/ansible/latest/dev_guide/testing/sanity/index.html)
- [ansible-test integration tests](https://docs.ansible.com/ansible/latest/dev_guide/testing_integration.html)
- [Collection structure](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_structure.html)
