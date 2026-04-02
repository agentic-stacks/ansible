# Performance Troubleshooting

Diagnose and resolve slow playbook runs, hanging tasks, and memory issues on the control node.

## Enable Profiling

The `profile_tasks` callback plugin shows execution time for every task:

```ini
# ansible.cfg
[defaults]
callbacks_enabled = ansible.posix.profile_tasks
```

Or via environment variable:

```bash
ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks ansible-playbook site.yml
```

Sample output:

```
Thursday 27 March 2026  14:22:10 +0000 (0:00:03.456)  0:01:23.789 ****
===============================================================================
Install packages -------------------------------------------------- 45.23s
Copy application files -------------------------------------------- 18.67s
Restart services -------------------------------------------------- 12.34s
Gather facts ------------------------------------------------------- 8.91s
```

Also useful: `profile_roles` (time per role) and `timer` (total playbook time).

## Common Causes Table

| Symptom | Likely Cause | Diagnostic | Fix |
|---|---|---|---|
| All tasks slow across all hosts | Low forks value | `ansible-config dump \| grep FORKS` | Increase `forks` in `ansible.cfg` (default is 5) |
| All tasks slow across all hosts | Pipelining disabled | `ansible-config dump \| grep PIPELINING` | Set `pipelining = True` in `[ssh_connection]` |
| One specific task is slow | Module or command doing heavy work | Run task with `-vvv`, check timing | Optimize the task, use async, or accept the time |
| Gathering facts is slow | Full fact gathering on every play | Check `gather_facts` and `gathering` setting | Use `gathering = smart` and enable fact caching |
| Serial on many hosts is slow | `serial: 1` or small serial value | Check play `serial` setting | Increase serial value or remove if not needed |
| Second play in playbook is slow | Redundant fact gathering | Facts gathered again per play | Set `gather_facts: false` on subsequent plays |

## Optimizing Forks and Pipelining

### Forks

Forks controls how many hosts Ansible manages in parallel (default: 5):

```ini
# ansible.cfg
[defaults]
forks = 50
```

Or per-run:

```bash
ansible-playbook site.yml -f 50
```

Guidelines:
- Start with 20-50 for most environments
- Monitor control node CPU and memory when increasing
- Diminishing returns above 100-200 depending on hardware

### Pipelining

Pipelining reduces the number of SSH operations per task from multiple (connect, transfer module, execute) to one:

```ini
# ansible.cfg
[ssh_connection]
pipelining = True
```

Requirements:
- `requiretty` must be disabled in sudoers on managed nodes (or use `Defaults:user !requiretty`)
- Provides 2-10x speedup depending on network latency

### SSH Multiplexing

SSH multiplexing (ControlPersist) is enabled by default. Verify it is not disabled:

```ini
# ansible.cfg
[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s
```

## Fact Caching

Avoid re-gathering facts on every playbook run:

```ini
# ansible.cfg
[defaults]
gathering = smart          # only gather facts if not cached
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 3600   # seconds
```

Other cache backends:

```ini
# Redis
fact_caching = community.general.redis
fact_caching_connection = localhost:6379:0

# Memcached
fact_caching = community.general.memcached
fact_caching_connection = ['localhost:11211']
```

Alternatively, skip fact gathering entirely when not needed:

```yaml
- hosts: webservers
  gather_facts: false
  tasks:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

## Strategy Plugins

### Free Strategy

By default Ansible uses the `linear` strategy — all hosts must finish a task before the next begins. The `free` strategy lets each host proceed independently:

```yaml
- hosts: webservers
  strategy: free
  tasks:
    - name: Install packages
      apt:
        name: nginx
        state: present
```

Use `free` when tasks do not depend on cross-host coordination. Avoid when tasks rely on delegation or serial ordering.

### Mitogen Strategy

The Mitogen strategy plugin replaces SSH with a faster pure-Python transport:

```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

Mitogen can provide 2-7x speedup by eliminating repeated SSH sessions and Python interpreter startup.

## Hanging Tasks

### Decision Tree

```
Task hangs indefinitely?
├── Is it an interactive prompt?
│   ├── Package manager asking for confirmation → Add -y flag or use module param
│   ├── SSH asking for password/passphrase → Use ssh-agent or ansible_password
│   └── Script waiting for input → Redirect stdin: `command: script.sh < /dev/null`
├── Is it an SSH issue?
│   ├── SSH connection stalled → Check network, set timeout:
│   │   ansible_ssh_extra_args: "-o ConnectTimeout=10 -o ServerAliveInterval=15"
│   └── ControlPersist socket stale → Remove: rm -rf ~/.ansible/cp/
├── Is it a long-running command?
│   └── Use async with poll:
│       async: 3600    # max runtime in seconds
│       poll: 30       # check every 30 seconds (0 = fire and forget)
└── Check with -vvvv to see where it blocks
```

### Async for Long-Running Tasks

```yaml
# Run a task asynchronously with polling
- name: Run long database migration
  command: /opt/app/migrate.sh
  async: 3600      # allow up to 1 hour
  poll: 30         # check status every 30 seconds

# Fire and forget (do not wait)
- name: Start background job
  command: /opt/app/batch_job.sh
  async: 3600
  poll: 0
  register: batch_job

# Check on it later
- name: Wait for background job
  async_status:
    jid: "{{ batch_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30
```

### Task Timeouts

Set a timeout on individual tasks to prevent indefinite hangs:

```yaml
- name: Task with timeout
  command: /opt/app/possibly_slow.sh
  timeout: 300   # fail after 5 minutes
```

## Memory Issues on the Control Node

### Symptoms

- Control node runs out of memory during large playbook runs
- OOM killer terminates Ansible process
- Extreme swapping and slow performance

### Causes and Fixes

| Cause | Fix |
|---|---|
| Too many forks for available RAM | Reduce `forks` — each fork consumes memory for facts and variables |
| Large fact sets from many hosts | Enable fact caching to avoid holding all facts in memory |
| Huge variables (e.g., `slurp` of large files) | Avoid loading large files into variables; use `fetch` or `copy` instead |
| Many registered variables | Limit `register` usage; use `no_log: true` to reduce memory for sensitive data |

### Estimation

Each fork uses roughly 30-100 MB depending on fact size and variable complexity. For 1000 hosts with `forks: 50`, expect 1.5-5 GB of RAM on the control node.

```bash
# Monitor during a run
watch -n 1 'ps aux | grep ansible | head -5; free -h'
```

## Quick Wins Checklist

- [ ] Set `forks = 20` or higher (default 5 is very conservative)
- [ ] Enable `pipelining = True`
- [ ] Set `gathering = smart` with fact caching
- [ ] Use `gather_facts: false` in plays that do not need facts
- [ ] Remove unnecessary `serial` constraints
- [ ] Use `free` strategy where task ordering is not required
- [ ] Add `async` to known long-running tasks
- [ ] Profile with `profile_tasks` to identify the actual bottleneck before optimizing
