# Troubleshooting Skill

Diagnose and resolve common Ansible failures using symptom-based lookup, decision trees, and targeted debugging techniques.

## Symptom Index

| Symptom | Category | Go To |
|---|---|---|
| "Permission denied", "Authentication failed" | Connection | [connection.md](connection.md) |
| "Unreachable", "Connection timed out" | Connection | [connection.md](connection.md) |
| "Missing sudo password" | Connection / Become | [connection.md](connection.md) |
| "MODULE FAILURE", "module not found" | Execution | [execution.md](execution.md) |
| "undefined variable", "AnsibleUndefinedVariable" | Execution | [execution.md](execution.md) |
| "template error", "Jinja2 error" | Execution | [execution.md](execution.md) |
| Playbook takes too long | Performance | [performance.md](performance.md) |
| Tasks hang indefinitely | Performance | [performance.md](performance.md) |

## General Debugging

### Increase Verbosity

Use the `-v` flag with increasing levels for more detail:

```bash
ansible-playbook site.yml -v      # basic output
ansible-playbook site.yml -vv     # task input details
ansible-playbook site.yml -vvv    # connection-level debug (SSH commands)
ansible-playbook site.yml -vvvv   # adds connection plugin internals (raw SSH, sftp)
```

`-vvvv` is the most useful level for diagnosing connection problems — it shows the exact SSH commands Ansible executes.

### Enable Debug Mode

```bash
ANSIBLE_DEBUG=true ansible-playbook site.yml
```

This produces very verbose internal debugging output. Redirect to a file for easier analysis:

```bash
ANSIBLE_DEBUG=true ansible-playbook site.yml 2>debug.log
```

### Inspect Active Configuration

```bash
# Show only settings that differ from defaults
ansible-config dump --changed

# Show the full resolved configuration
ansible-config dump

# List all configuration options with descriptions
ansible-config list
```

### Check Environment

```bash
# Verify Ansible version and Python interpreter
ansible --version

# Test host connectivity
ansible all -m ping

# Show gathered facts for a host
ansible hostname -m setup
```

### Isolate Failures

When a playbook fails, narrow the scope:

```bash
# Run against a single host
ansible-playbook site.yml --limit problem-host

# Start from a specific task
ansible-playbook site.yml --start-at-task "task name"

# Step through tasks one at a time
ansible-playbook site.yml --step

# Check mode (dry run)
ansible-playbook site.yml --check --diff
```
