# Execution Strategy Selection

## Context

Strategy plugins control how Ansible executes tasks across the host list.
The strategy is set per play with the `strategy:` directive and defaults to `linear`.

## Comparison Table

| Strategy | Execution Model | Use Case | Risk |
|---|---|---|---|
| `linear` | One task at a time across all hosts; waits for every host to finish before the next task | Default for most playbooks; safe rolling updates | Slowest option when hosts vary in speed |
| `free` | Each host runs tasks as fast as possible without waiting for others | Independent hosts (e.g., stateless web servers) where order does not matter | Task ordering across hosts is unpredictable; harder to debug |
| `host_pinned` | Like `free`, but keeps the worker pinned to a host until its play finishes | Long-running plays where connection setup is expensive | Same ordering unpredictability as `free` |
| `mitogen` | Replaces SSH-per-task with a persistent Python interpreter on each host; dramatically faster | Any playbook where SSH overhead dominates (large host counts, many small tasks) | Third-party plugin; requires install (`pip install mitogen`); not supported by Red Hat |

## Recommendation

| Scenario | Strategy |
|---|---|
| Default / first choice | `linear` — safe, predictable, easiest to debug |
| Speed matters and hosts are independent | `free` — removes the per-task synchronization barrier |
| Speed matters and task ordering matters | `mitogen` — keeps linear semantics but cuts SSH overhead |

## Migration Path

Changing strategy is low risk:

1. Set `strategy:` at the **play level** to test on a single play first.
2. Validate idempotency by running twice — results should be unchanged.
3. Expand to remaining plays once confident.

```yaml
# Example: test free strategy on one play
- name: Deploy app servers
  hosts: app
  strategy: free
  tasks:
    - name: Install packages
      ansible.builtin.dnf:
        name: "{{ packages }}"
        state: present
```
