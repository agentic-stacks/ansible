# Creating Collections

## Scaffold a New Collection

Use `ansible-galaxy collection init` to generate the standard directory layout.

```bash
# Create the skeleton
ansible-galaxy collection init my_namespace.my_collection

# Create inside a specific directory
ansible-galaxy collection init my_namespace.my_collection --init-path ./collections/ansible_collections
```

This produces a ready-to-use structure with `galaxy.yml`, placeholder directories, and a default README.

## Collection Structure

```
my_namespace/my_collection/
├── galaxy.yml                # Collection metadata — namespace, name, version, dependencies
├── README.md                 # Collection documentation
├── meta/
│   └── runtime.yml           # Redirect and deprecation mappings
├── plugins/
│   ├── modules/              # Custom modules
│   │   └── my_module.py
│   ├── inventory/            # Inventory plugins
│   │   └── my_inventory.py
│   ├── lookup/               # Lookup plugins
│   ├── filter/               # Filter plugins
│   ├── callback/             # Callback plugins
│   └── module_utils/         # Shared Python code for modules
├── roles/
│   └── my_role/
│       ├── tasks/
│       │   └── main.yml
│       ├── defaults/
│       │   └── main.yml
│       └── meta/
│           └── main.yml
├── playbooks/                # Playbooks shipped with the collection
├── docs/                     # Additional documentation
└── tests/                    # Integration and unit tests
```

### galaxy.yml

The `galaxy.yml` file defines metadata for building and publishing.

```yaml
# galaxy.yml
---
namespace: my_namespace
name: my_collection
version: 1.0.0
readme: README.md
authors:
  - Your Name <you@example.com>
description: A short description of the collection.
license:
  - GPL-2.0-or-later
repository: https://github.com/example/my_namespace.my_collection
dependencies:
  ansible.netcommon: ">=5.0.0"
  community.general: ">=8.0.0"
build_ignore:
  - .gitignore
  - .github
  - tests/output
```

## Build and Install Locally

### Build the tarball

```bash
cd my_namespace/my_collection/

# Build produces a .tar.gz in the current directory
ansible-galaxy collection build

# Output: Created collection for my_namespace.my_collection at
#   ./my_namespace-my_collection-1.0.0.tar.gz
```

### Install from the tarball

```bash
# Install system-wide
ansible-galaxy collection install ./my_namespace-my_collection-1.0.0.tar.gz

# Install to a project-local path
ansible-galaxy collection install ./my_namespace-my_collection-1.0.0.tar.gz -p ./collections/
```

### Verify the installation

```bash
ansible-galaxy collection list | grep my_namespace.my_collection
```

## Publish to Galaxy

### Set up your API key

1. Log in to [Ansible Galaxy](https://galaxy.ansible.com/) and retrieve your API token from **Preferences > API Key**.
2. Store the token so `ansible-galaxy` can use it:

```bash
# Option 1: Pass the token on the command line
ansible-galaxy collection publish ./my_namespace-my_collection-1.0.0.tar.gz --api-key YOUR_API_KEY

# Option 2: Configure the token in ansible.cfg
# ansible.cfg
# [galaxy]
# server_list = release_galaxy
#
# [galaxy_server.release_galaxy]
# url = https://galaxy.ansible.com/
# token = YOUR_API_KEY
```

### Publish the collection

```bash
# Build first, then publish
ansible-galaxy collection build
ansible-galaxy collection publish ./my_namespace-my_collection-1.0.0.tar.gz
```

To publish to Automation Hub instead of Galaxy, set the server URL accordingly:

```ini
# ansible.cfg
[galaxy_server.automation_hub]
url = https://console.redhat.com/api/automation-hub/
auth_url = https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token = YOUR_AUTOMATION_HUB_TOKEN
```

## meta/runtime.yml

The `meta/runtime.yml` file defines module redirects, deprecations, and the minimum required version of ansible-core.

```yaml
# meta/runtime.yml
---
requires_ansible: ">=2.15.0"

plugin_routing:
  modules:
    # Redirect an old module name to its new location
    old_module_name:
      redirect: my_namespace.my_collection.new_module_name

    # Deprecate a module with a removal target
    deprecated_module:
      deprecation:
        removal_version: "3.0.0"
        warning_text: >
          deprecated_module has been deprecated.
          Use new_module instead.

    # Redirect to a module in another collection
    moved_module:
      redirect: other_namespace.other_collection.the_module

  lookup:
    old_lookup:
      redirect: my_namespace.my_collection.new_lookup
```

### Common use cases

| Routing Action | Purpose |
|---------------|---------|
| `redirect` | Point an old plugin name to its new name, within this collection or another |
| `deprecation` with `removal_version` | Warn users that a plugin will be removed in a future version |
| `deprecation` with `removal_date` | Warn users with a calendar-based removal target (e.g., `"2025-12-01"`) |
| `tombstone` | Mark a plugin as removed; Ansible raises an error if it is used |

## References

- [Developing Collections](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html)
- [Creating Collections](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_creating.html)
- [Collection Structure](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_structure.html)
- [Distributing Collections](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_distributing.html)
