# ansible-lint

ansible-lint checks playbooks, roles, and collections for best practices, common mistakes, and style consistency. It catches problems statically before any infrastructure is touched.

## Install

```bash
pip install ansible-lint
```

```bash
# Verify installation
ansible-lint --version
```

## Run

Lint playbooks and roles by passing paths:

```bash
# Lint specific directories
ansible-lint playbooks/ roles/

# Lint a single playbook
ansible-lint playbooks/site.yml

# Parseable output for CI (one finding per line)
ansible-lint -p playbooks/ roles/

# Output as JSON for tooling integration
ansible-lint -f json playbooks/

# Show only specific severity levels
ansible-lint --strict playbooks/
```

When run with no arguments, ansible-lint looks for playbooks and roles in the current directory.

## Profiles

Profiles bundle sets of rules at increasing strictness levels. Set a profile to enforce a baseline without listing individual rules.

| Profile | Strictness | Description |
|---------|-----------|-------------|
| `min` | Lowest | Only critical rules (syntax errors, deprecated modules) |
| `basic` | Low | Adds command-shell best practices, FQCN requirements |
| `moderate` | Medium | Adds formatting, naming conventions, task hygiene |
| `safety` | High | Adds security-related rules (no plaintext secrets, proper permissions) |
| `shared` | Higher | Rules for code shared across teams or published to Galaxy |
| `production` | Highest | All rules enabled, strictest enforcement |

Set the profile in `.ansible-lint`:

```yaml
profile: moderate
```

Or on the command line:

```bash
ansible-lint --profile moderate playbooks/
```

Start with `basic` or `moderate` and increase over time as the codebase matures.

## Configuration

Create a `.ansible-lint` file in the project root:

```yaml
---
profile: moderate

# Paths to exclude from linting
exclude_paths:
  - .cache/
  - .github/
  - molecule/
  - tests/
  - collections/

# Rules to skip entirely (never report)
skip_list:
  - yaml[line-length]
  - name[template]

# Rules to report as warnings (non-blocking in CI)
warn_list:
  - role-name[path]
  - experimental

# Rules to enable that are disabled by default
enable_list:
  - no-log-password
  - no-same-owner

# Offline mode — skip installing dependencies
offline: false

# Use progressive mode — only report new violations
use_default_rules: true

# Kinds lets you tell ansible-lint the type of files
kinds:
  - playbook: "**/playbooks/*.yml"
  - tasks: "**/tasks/*.yml"
  - vars: "**/vars/*.yml"
  - handlers: "**/handlers/*.yml"
```

## Common Rules

Frequently triggered rules and how to fix them:

| Rule ID | Description | Fix |
|---------|-------------|-----|
| `yaml[truthy]` | Truthy values should be `true`/`false` | Replace `yes`/`no` with `true`/`false` |
| `yaml[line-length]` | Line exceeds max length (160 chars) | Break long lines or use `>` folded scalar |
| `fqcn[action-core]` | Use FQCN for built-in modules | Change `copy:` to `ansible.builtin.copy:` |
| `fqcn[action]` | Use FQCN for non-built-in modules | Change `docker_container:` to `community.docker.docker_container:` |
| `name[missing]` | Tasks should have a `name:` | Add a descriptive `name:` to the task |
| `name[casing]` | Task names should start with uppercase | Capitalize the first letter of task names |
| `name[template]` | Task name should not use Jinja2 | Remove `{{ }}` from task names; use static names |
| `no-changed-when` | Commands/shell tasks need `changed_when` | Add `changed_when: false` or a proper condition |
| `risky-file-permissions` | Missing mode on file/copy/template | Add `mode: "0644"` (or appropriate permissions) |
| `command-instead-of-module` | Use a module instead of command/shell | Replace `command: apt install` with `ansible.builtin.apt:` |
| `no-jinja-when` | `when:` already implies Jinja2 | Remove `{{ }}` from `when:` conditions |
| `role-name[path]` | Role name does not match path | Rename the role directory to match `meta/main.yml` |
| `deprecated-module` | Module is deprecated | Switch to the replacement module listed in docs |
| `key-order[task]` | Task keys should follow standard order | Reorder: `name`, `when`, `tags`, `become`, then module |

## Custom Rules

Write custom rules as Python classes in a rules directory:

```python
# custom_rules/no_debug_in_prod.py
from ansiblelint.rules import AnsibleLintRule


class NoDebugInProd(AnsibleLintRule):
    """Disallow ansible.builtin.debug tasks in production playbooks."""

    id = "custom-no-debug"
    shortdesc = "Debug tasks should not be in production playbooks"
    description = (
        "Tasks using ansible.builtin.debug are for development only. "
        "Remove them before merging to main."
    )
    severity = "MEDIUM"
    tags = ["custom", "production"]

    def matchtask(self, task, file=None):
        if task["action"]["__ansible_module__"] == "ansible.builtin.debug":
            return self.shortdesc
        return False
```

Tell ansible-lint where to find custom rules:

```yaml
# .ansible-lint
rulesdir:
  - custom_rules/
```

Or on the command line:

```bash
ansible-lint -R -r custom_rules/ playbooks/
```

## CI Integration

### pre-commit Hook

Add ansible-lint to your `.pre-commit-config.yaml`:

```yaml
---
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v25.1.0
    hooks:
      - id: ansible-lint
        additional_dependencies:
          - ansible
```

```bash
# Install and run
pip install pre-commit
pre-commit install
pre-commit run ansible-lint --all-files
```

### GitHub Actions

```yaml
# .github/workflows/lint.yml
---
name: Ansible Lint
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install ansible-lint
        run: pip install ansible-lint

      - name: Run ansible-lint
        run: ansible-lint --profile moderate -p playbooks/ roles/
```

The `-p` flag produces parseable output that integrates with CI log parsers and GitHub annotations.

## References

- [ansible-lint documentation](https://ansible.readthedocs.io/projects/lint/)
- [ansible-lint rules reference](https://ansible.readthedocs.io/projects/lint/rules/)
- [ansible-lint profiles](https://ansible.readthedocs.io/projects/lint/profiles/)
- [ansible-lint configuration](https://ansible.readthedocs.io/projects/lint/configuring/)
