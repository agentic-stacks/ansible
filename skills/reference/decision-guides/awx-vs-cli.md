# AWX/AAP vs CLI

## Context

- **AWX** is the open-source upstream project that provides a web UI, REST API, and RBAC layer on top of Ansible.
- **AAP** (Ansible Automation Platform) is the Red Hat enterprise product built from AWX, adding vendor support, certified content, and additional tooling.
- **CLI** refers to running `ansible-playbook` directly from a shell, cron job, or CI/CD pipeline.

## Comparison Table

| Feature | CLI | AWX / AAP |
|---|---|---|
| Cost | Free | AWX free; AAP requires subscription |
| UI | None (terminal only) | Web dashboard with job visualization |
| RBAC | OS-level permissions only | Fine-grained role-based access control |
| Scheduling | Cron or CI/CD scheduler | Built-in job scheduling |
| Audit trail | Manual logging | Automatic job history and output capture |
| Credential management | Vault files, env vars | Encrypted credential store with injection |
| API | None built-in | Full REST API for every operation |
| CI/CD integration | Native (runs in any pipeline) | Requires API calls or collection modules |
| Complexity | Low — single binary + inventory | Medium-High — requires PostgreSQL, Redis, containers |
| Scaling | Manual (SSH multiplexing, forks) | Built-in instance groups and capacity management |

## Recommendation by Use Case

| Scenario | Recommendation |
|---|---|
| Small team, fewer than 50 hosts | **CLI** — low overhead, fast iteration |
| Team of 5+ operators, compliance requirements | **AWX** — RBAC and audit trail justify the setup cost |
| Enterprise environment, vendor support needed | **AAP** — certified content, SLA-backed support |
| CI/CD-heavy workflow, GitOps model | **CLI** — runs natively in any pipeline without an extra service |

## Future Stack

Full AWX/AAP coverage is planned in the `ansible-awx` stack.
See the stack-factory queue for status: <https://github.com/agentic-stacks/stack-factory/blob/main/queue.yaml>
