# Scale Profile

## Overview
Ansible configuration optimized for managing large fleets (hundreds to thousands of nodes) with maximum throughput and minimal control-node overhead.

## Recommended Settings
| Setting | Value | Why |
|---------|-------|-----|
| `forks` | `50` | Parallelism — number of simultaneous host connections |
| `pipelining` | `True` | Reduces SSH round-trips by executing modules without temporary files |
| `gathering` | `smart` | Cache facts and only re-gather when missing |
| `fact_caching` | `redis` | Persistent, shared fact cache across playbook runs |
| `strategy` | `free` | Hosts proceed independently instead of waiting at each task |

### ansible.cfg Example
```ini
[defaults]
forks = 50
gathering = smart
fact_caching = redis
fact_caching_connection = localhost:6379:0
fact_caching_timeout = 86400
strategy = free

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

## ansible-pull for 1000+ Nodes
At extreme scale (1000+ managed nodes), switch from push mode to `ansible-pull`:
- Each node pulls its own configuration from a Git repository on a cron schedule.
- Eliminates the control-node bottleneck entirely.
- Combine with `fact_caching = redis` so nodes share a central fact store.

```bash
# Example cron entry on each node
*/15 * * * * ansible-pull -U https://git.example.com/ansible-config.git -C main site.yml
```

## Applicable Skills
- `skills/foundation/configuration` — tuning ansible.cfg for high-fork, pipelined execution
- `skills/operations/performance` — profiling callback plugins, bottleneck identification
- `skills/deploy/inventory` — dynamic inventory plugins for cloud-scale environments
- `skills/operations/testing` — validating playbook runs at scale with `--check` and `--diff`
