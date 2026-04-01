# ansible

An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that teaches AI agents how to operate Ansible.

Covers ansible-core 2.16–2.17 and Ansible community package 9.x–10.x.

## Skills

| Phase | Skills |
|---|---|
| Foundation | concepts, installation, configuration |
| Deploy | inventory, playbooks, roles, collections, modules |
| Operations | vault, testing, performance, upgrades |
| Diagnose | troubleshooting |
| Reference | known-issues, decision-guides, compatibility |

## Profiles

| Profile | Focus |
|---|---|
| security | Hardened automation, vault-first |
| networking | Network device automation (IOS, EOS, JUNOS) |
| storage | LVM, NFS, iSCSI, ZFS management |
| scale | Large fleet optimization (1000+ nodes) |
| features | Bleeding-edge ansible-core 2.17 features |

## Usage

```bash
agentic-stacks init agentic-stacks/ansible my-project
cd my-project
agentic-stacks pull
```

Then start Claude Code — it reads `.stacks/ansible/CLAUDE.md` and becomes an expert operator.
