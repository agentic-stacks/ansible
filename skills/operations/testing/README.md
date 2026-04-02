# Testing

Testing strategy overview for Ansible projects. Choose the right tool based on what you need to test.

## Tool Comparison

| Tool | Tests What | When to Use |
|------|-----------|-------------|
| [Molecule](molecule.md) | Roles end-to-end in real containers | Role development, convergence and idempotence checks |
| [ansible-lint](ansible-lint.md) | Playbooks, roles, collections for style and best practices | Every commit, pre-commit hooks, CI gates |
| [ansible-test](ansible-test.md) | Collection sanity, units, and integration | Collection development only (not standalone roles/playbooks) |

## Quick Start

```bash
# Lint everything
ansible-lint playbooks/ roles/

# Test a role with Molecule
cd roles/my_role
molecule test

# Run collection sanity checks
cd ~/.ansible/collections/ansible_collections/my_namespace/my_collection
ansible-test sanity
```

## Recommended Test Pipeline

Run these in order during CI:

1. **Lint** — `ansible-lint` catches style issues and common mistakes before any infrastructure is created.
2. **Molecule** — `molecule test` converges roles in containers and verifies the result.
3. **ansible-test** — `ansible-test sanity && ansible-test units && ansible-test integration` validates collections.

```bash
# Combined CI script
ansible-lint playbooks/ roles/
cd roles/my_role && molecule test
cd collections/ansible_collections/ns/name && ansible-test sanity --docker default
```

## References

- [Molecule documentation](https://ansible.readthedocs.io/projects/molecule/)
- [ansible-lint documentation](https://ansible.readthedocs.io/projects/lint/)
- [ansible-test documentation](https://docs.ansible.com/ansible/latest/dev_guide/testing.html)
