# Role Creation

## Scaffold with ansible-galaxy

Generate a complete role skeleton with a single command.

```bash
ansible-galaxy role init my_role
```

Resulting directory tree:

```
my_role/
├── README.md
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml
```

Scaffold into a custom path:

```bash
# Create inside an existing roles/ directory
ansible-galaxy role init my_role --init-path ./roles/

# Use an offline skeleton (no Galaxy metadata)
ansible-galaxy role init my_role --offline
```

## Create a Minimum Viable Role

A role only requires `tasks/main.yml`. All other directories are optional.

```
my_role/
└── tasks/
    └── main.yml
```

```yaml
# tasks/main.yml
---
- name: Install required packages
  ansible.builtin.apt:
    name: "{{ my_role_packages }}"
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"

- name: Install required packages (RedHat)
  ansible.builtin.dnf:
    name: "{{ my_role_packages }}"
    state: present
  when: ansible_os_family == "RedHat"
```

Add directories only when the role needs them. An empty `handlers/` or `vars/` directory adds no value.

## Follow Best Practices

### Keep single responsibility

Each role should do one thing well. Prefer composing multiple small roles over one monolithic role.

| Approach | Example |
|----------|---------|
| Good | `nginx`, `app_deploy`, `ssl_certs` as separate roles |
| Bad | `web_server` that installs nginx, deploys the app, and manages certificates |

### Parameterize with defaults

Expose every user-configurable value through `defaults/main.yml`. Prefix variable names with the role name to avoid collisions.

```yaml
# defaults/main.yml
---
myapp_version: "1.0.0"
myapp_install_dir: /opt/myapp
myapp_port: 8080
myapp_user: myapp
myapp_group: myapp
myapp_log_level: info
myapp_enable_ssl: false
myapp_ssl_cert_path: /etc/ssl/certs/myapp.pem
myapp_ssl_key_path: /etc/ssl/private/myapp.key
```

### Validate inputs early

Add assertions at the top of `tasks/main.yml` to fail fast with clear messages.

```yaml
# tasks/main.yml
---
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - myapp_version is defined
      - myapp_port | int > 0
      - myapp_port | int < 65536
    fail_msg: "Required variable missing or invalid. Check defaults/main.yml for expected variables."

- name: Validate SSL configuration
  ansible.builtin.assert:
    that:
      - myapp_ssl_cert_path is defined
      - myapp_ssl_key_path is defined
    fail_msg: "SSL is enabled but certificate paths are not configured."
  when: myapp_enable_ssl
```

### Use fully qualified collection names

Always use FQCNs for modules to avoid ambiguity.

```yaml
# Good
- name: Install package
  ansible.builtin.apt:
    name: nginx

# Bad
- name: Install package
  apt:
    name: nginx
```

### Add a molecule test scaffold

Initialize Molecule for the role to enable automated testing.

```bash
cd my_role/
molecule init scenario --driver-name docker
```

See `skills/operations/testing` for Molecule setup and test configuration.

### Document the role

Include a `README.md` at the role root with:

- Role description and purpose
- Requirements (minimum Ansible version, OS support)
- Role variables table with names, defaults, and descriptions
- Example playbook usage
- License and author

```markdown
# my_role

Install and configure MyApp.

## Requirements

- Ansible >= 2.15
- Supported OS: Ubuntu 22.04, RHEL 9

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `myapp_version` | `1.0.0` | Application version to install |
| `myapp_port` | `8080` | Listen port |
| `myapp_enable_ssl` | `false` | Enable TLS termination |

## Example Playbook

    - hosts: app_servers
      roles:
        - role: my_role
          vars:
            myapp_version: "2.0.0"
            myapp_port: 9090

## License

MIT
```

## Publish to Galaxy

### Configure Galaxy metadata

Populate `meta/main.yml` with required fields.

```yaml
# meta/main.yml
---
galaxy_info:
  author: your_name
  description: Install and configure MyApp
  company: Example Inc

  license: MIT

  min_ansible_version: "2.15"

  platforms:
    - name: Ubuntu
      versions:
        - jammy
    - name: EL
      versions:
        - "9"

  galaxy_tags:
    - myapp
    - web
    - deployment

dependencies: []
```

| Field | Required | Description |
|-------|----------|-------------|
| `author` | Yes | Author name or GitHub username |
| `description` | Yes | Short role description |
| `license` | Yes | SPDX license identifier |
| `min_ansible_version` | Recommended | Minimum supported Ansible version |
| `platforms` | Recommended | List of supported OS and versions |
| `galaxy_tags` | Recommended | Tags for Galaxy search (max 20) |
| `company` | No | Organization name |

### Import to Galaxy

```bash
# Log in to Galaxy (requires GitHub token)
ansible-galaxy login --github-token YOUR_TOKEN

# Import a role from a GitHub repository
ansible-galaxy role import github_user repo_name

# Import with a specific role name
ansible-galaxy role import github_user repo_name --role-name my_role

# Import a specific branch
ansible-galaxy role import github_user repo_name --branch main
```

The repository name on GitHub should match the pattern `ansible-role-<role_name>`. Galaxy strips the `ansible-role-` prefix automatically.

### Version releases

Tag releases in git to publish specific versions to Galaxy.

```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```

Galaxy detects tags and makes them available as installable versions. Users can then pin to a specific version in their `requirements.yml`.

## Test a Role

Verify role behavior with Molecule before publishing.

```bash
# Run the full test sequence (lint, create, converge, verify, destroy)
molecule test

# Run only converge for iterative development
molecule converge

# Log in to the test instance for debugging
molecule login
```

See `skills/operations/testing` for Molecule setup, driver configuration, and CI integration.

## References

- [Creating Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-directory-structure)
- [ansible-galaxy role init](https://docs.ansible.com/ansible/latest/cli/ansible-galaxy.html)
- [Ansible Galaxy — Publishing Roles](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#importing-roles)
- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
