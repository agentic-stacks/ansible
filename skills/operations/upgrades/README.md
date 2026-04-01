# Upgrades

## Before You Upgrade

> **Warning** Always check `skills/reference/known-issues/` for the target version first.

Before upgrading any Ansible component, complete these steps:

1. Read the changelog for the target version.
2. Run your existing playbooks with `-v` to surface deprecation warnings.
3. Test the upgrade in a staging environment before touching production.

```bash
# Surface deprecation warnings in your playbooks
ansible-playbook site.yml --check -v 2>&1 | grep -i "DEPRECATION WARNING"
```

Skipping these steps risks breaking production playbooks with removed features or changed behavior.

## Release Cadence

**ansible-core** releases follow a graduated schedule:

- New major version (e.g., 2.17 to 2.18): approximately every 6 months.
- Minor/patch releases (e.g., 2.18.1): every four weeks for the latest three major versions.
- Security-only fixes continue for the oldest of the three maintained versions.

**Ansible community package** follows ansible-core:

- New major version (e.g., 12.x to 13.x) ships after each new ansible-core major release.
- Minor releases every four weeks, bundling updated collections.
- Only one major version is maintained at a time. When 13.0.0 releases, 12.x reaches end of life.

The current mapping (as of early 2026):

| Ansible Package | ansible-core | Status           |
|-----------------|--------------|------------------|
| 14.x            | 2.21         | In development   |
| 13.x            | 2.20         | Current (latest) |
| 12.x            | 2.19         | EOL              |

## Upgrade ansible-core (Minor Version)

Minor version upgrades within the same major line (e.g., 2.20.1 to 2.20.3) are low-risk. They contain bug fixes and security patches only.

```bash
# Upgrade to latest patch within 2.20
pip install --upgrade 'ansible-core>=2.20,<2.21'

# Or pin to a specific patch
pip install ansible-core==2.20.3
```

**What can break:**

- Rarely, a bug fix changes behavior you relied on.
- New Python version requirements are possible but uncommon in patch releases.

After upgrading, run your test playbooks to confirm nothing changed unexpectedly.

## Upgrade ansible-core (Major Version)

Major version upgrades (e.g., 2.19 to 2.20) can remove deprecated features and change defaults. Follow this step-by-step process:

### Step 1 — Read the Porting Guide

Every major release has a porting guide that lists breaking changes, removed features, and required actions.

- ansible-core 2.16: https://docs.ansible.com/ansible-core/2.16/porting_guides/porting_guide_core_2.16.html
- ansible-core 2.17: https://docs.ansible.com/ansible-core/2.17/porting_guides/porting_guide_core_2.17.html

Substitute your target version in the URL pattern: `porting_guide_core_2.XX.html`.

### Step 2 — Fix Deprecation Warnings

Run all playbooks with verbosity and address every deprecation warning before upgrading:

```bash
ansible-playbook site.yml --check -v 2>&1 | grep "DEPRECATION WARNING"
```

Each warning tells you what to use instead. Fix these on your current version before upgrading.

### Step 3 — Update Python If Needed

Each ansible-core release supports the three most recently released Python versions on the control node. Check that your Python version is supported:

```bash
python3 --version
ansible --version  # Shows Python path and version
```

If your Python is too old, upgrade it before upgrading ansible-core.

### Step 4 — Perform the Upgrade

```bash
pip install ansible-core==2.20.*
```

### Step 5 — Test

Run your full playbook suite in staging. Pay attention to:

- Tasks that previously showed deprecation warnings.
- Any modules that changed behavior or moved to a different collection.
- Variable precedence or templating changes (especially in 2.19+).

## Upgrade Ansible Community Package

The Ansible community package bundles ansible-core plus a curated set of collections.

```bash
# Upgrade to latest Ansible 13.x
pip install --upgrade 'ansible>=13,<14'

# Or pin a specific version
pip install ansible==13.1.0
```

After upgrading, verify collection compatibility:

```bash
# List all installed collections and their versions
ansible-galaxy collection list

# Check for any collection conflicts
pip check
```

If a collection included in the Ansible package conflicts with a separately installed version, remove the standalone version and rely on the bundled one.

## Handle Deprecation Warnings

Deprecation warnings tell you what will break in a future release. Treat them as required work, not noise.

### Enable Deprecation Warnings

Ensure warnings are visible in `ansible.cfg`:

```ini
[defaults]
deprecation_warnings = True
```

This is the default, but some teams disable it. Turn it back on before any upgrade planning.

### Find All Warnings

```bash
# Run playbooks with verbosity to surface warnings
ansible-playbook site.yml -v 2>&1 | grep -i "DEPRECATION WARNING"

# Each warning follows this pattern:
# [DEPRECATION WARNING]: Using <old_thing> is deprecated. Use <new_thing> instead.
# This feature will be removed in version X.Y.
```

The warning text tells you exactly what to change and which version will remove the feature. Fix deprecations targeting your next upgrade version first.

### Common Deprecation Patterns

- **Module renamed**: `old_module` replaced by `namespace.collection.new_module`. Update the `module:` line in your tasks.
- **Parameter removed**: A module parameter is going away. The warning names the replacement parameter.
- **Bare variables in conditionals**: Use `variable` not `{{ variable }}` in `when:` clauses.
- **Templating changes**: ansible-core 2.19 introduced stricter templating that flags previously silent issues.

## Porting Guides

Porting guides are the definitive reference for upgrade-breaking changes. Bookmark this index:

```
https://docs.ansible.com/ansible/latest/porting_guides/
```

Each guide covers:

- Playbook changes required (new syntax, removed parameters).
- Module and plugin changes (moved, renamed, removed).
- Command-line changes.
- Python version requirements.
- Deprecated features scheduled for removal.

Read the porting guide for every major version between your current and target version. If you are jumping from ansible-core 2.17 to 2.20, read the guides for 2.18, 2.19, and 2.20.

## Pin and Freeze

After a successful upgrade, record exactly what you have installed. This creates an audit trail and makes rollbacks possible.

```bash
# Freeze Python packages (ansible-core and all dependencies)
pip freeze > requirements.txt

# Record installed collections
ansible-galaxy collection list > collections-manifest.txt
```

Store both files in version control. When you need to reproduce the environment:

```bash
pip install -r requirements.txt
```

For collections, use a `requirements.yml` for reproducible installs:

```yaml
# requirements.yml
collections:
  - name: community.general
    version: ">=8.0.0,<9.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
```

This ensures every team member and CI pipeline uses the same versions.
