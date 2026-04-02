# Performance

## Quick Wins

Three settings that deliver the largest speedup with minimal risk.

| Setting | Default | Tuned | Effect |
|---------|---------|-------|--------|
| `forks` | `5` | `20`-`50` | Number of parallel host connections. Increase to match your control node CPU/RAM. |
| `pipelining` | `False` | `True` | Reduces SSH operations per task from multiple to one. Requires `requiretty` disabled in sudoers. |
| `gathering` | `implicit` | `smart` | Caches facts and only gathers when cache is empty or expired. |

Apply all three in `ansible.cfg`:

```ini
[defaults]
forks = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400

[ssh_connection]
pipelining = True
```

> **Warning** Setting `forks` too high (100+) can overwhelm the control node or trigger connection limits on targets. Profile before tuning beyond 50.

## ansible.cfg Performance Settings

### [defaults] Section

```ini
[defaults]
# Parallelism
forks = 30

# Strategy — linear is default; free lets hosts run independently
strategy = linear

# Fact gathering
gathering = smart
gather_subset = !hardware,!facter,!ohai
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400

# Callback plugins — enable profiling
callbacks_enabled = timer, profile_tasks, profile_roles

# Reduce output for faster display
stdout_callback = yaml

# Retry files clutter the project — disable
retry_files_enabled = False

# Poll interval for async tasks (seconds)
poll_interval = 5

# Disable host key checking for dynamic environments
host_key_checking = False
```

### [ssh_connection] Section

```ini
[ssh_connection]
# Combine multiple SSH operations into a single connection
pipelining = True

# SSH multiplexing arguments (see SSH Multiplexing section)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/tmp/ansible-ssh-%h-%p-%r

# SCP vs SFTP — sftp is default and preferred
transfer_method = sftp

# Number of ssh connection retries
retries = 3
```

## SSH Multiplexing

SSH multiplexing reuses a single TCP connection for multiple SSH sessions to the same host, eliminating the handshake overhead for each task.

The three critical OpenSSH options:

| Option | Value | Purpose |
|--------|-------|---------|
| `ControlMaster` | `auto` | Automatically create a master connection if none exists. |
| `ControlPath` | `/tmp/ansible-ssh-%h-%p-%r` | Socket file path. Use `%h` (host), `%p` (port), `%r` (user) to avoid collisions. |
| `ControlPersist` | `60s` | Keep the master connection open for 60 seconds after the last session closes. |

Set these in `ansible.cfg`:

```ini
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/tmp/ansible-ssh-%h-%p-%r
```

> **Note** If you see `ControlPath too long` errors, shorten the path. Some systems limit Unix socket paths to 104 characters. Use a shorter directory or hash the hostname:
>
> ```ini
> ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/tmp/a-%r@%h:%p
> ```

## Strategy Plugins

A strategy plugin controls how tasks are dispatched across hosts.

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `linear` (default) | All hosts execute task N before any host starts task N+1. | Ordered rollouts, dependencies between tasks. |
| `free` | Each host runs the entire play as fast as it can, independently. | Heterogeneous hosts, no inter-host dependencies. |
| `host_pinned` | Like `free`, but each worker stays pinned to a host until its play completes. | Consistent resource allocation, fewer context switches. |

Set the strategy per play or globally:

```yaml
# Per play
- hosts: webservers
  strategy: free
  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: nginx
        state: present
```

```ini
# Global default in ansible.cfg
[defaults]
strategy = free
```

Use `serial` with the `linear` strategy to process hosts in batches (rolling updates):

```yaml
- hosts: webservers
  serial: 5
  tasks:
    - name: Deploy application
      ansible.builtin.copy:
        src: app.tar.gz
        dest: /opt/app/
```

> Cross-reference: `skills/reference/decision-guides/strategy-selection.md`

## Fact Caching

Fact caching stores `gather_facts` output so subsequent playbook runs skip the gathering phase entirely.

### jsonfile Backend

File-based cache. No external dependencies. Good for single control nodes.

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400
```

Each host gets one JSON file in the connection directory:

```
/tmp/ansible_fact_cache/
  webserver01
  webserver02
  dbserver01
```

### Redis Backend

Shared cache for multiple control nodes or CI runners. Requires the `redis` Python package.

```bash
pip install redis
```

```ini
[defaults]
gathering = smart
fact_caching = redis
fact_caching_connection = localhost:6379:0
fact_caching_timeout = 86400
```

The connection string format is `host:port:database`. To add a password:

```ini
fact_caching_connection = localhost:6379:0:my_redis_password
```

### Cache Timeout

`fact_caching_timeout` is in seconds. Common values:

| Value | Duration | Use Case |
|-------|----------|----------|
| `3600` | 1 hour | Frequently changing infrastructure. |
| `86400` | 24 hours | Stable environments, daily runs. |
| `0` | Never expires | Static inventory that rarely changes. |

Set `gathering = smart` so Ansible checks the cache first and only gathers facts when the cache is missing or expired.

## Profiling Slow Playbooks

The `profile_tasks` callback plugin prints execution time per task, making it easy to find bottlenecks.

### Enable Profiling

```ini
[defaults]
callbacks_enabled = timer, profile_tasks, profile_roles
```

Or set the environment variable:

```bash
ANSIBLE_CALLBACKS_ENABLED=timer,profile_tasks,profile_roles ansible-playbook site.yml
```

### Reading the Output

After a playbook run, `profile_tasks` appends a summary sorted by duration:

```
Tuesday 31 March 2026  14:22:01 +0000 (0:00:01.234)  0:02:15.678 ********

Playbook run took 0 days, 0 hours, 2 minutes, 15 seconds

 Install packages ------------------------------------------ 45.23s
 Configure nginx ------------------------------------------- 30.12s
 Gather facts ---------------------------------------------- 18.45s
 Copy application files ------------------------------------ 12.89s
 Restart services ------------------------------------------  5.67s
```

### Identifying Bottlenecks

Common patterns and fixes:

| Bottleneck | Symptom | Fix |
|------------|---------|-----|
| Slow fact gathering | `Gather facts` high in profile | Use `gathering: smart` + fact caching, or `gather_facts: false` if facts are not needed. |
| Package installs | Individual `apt`/`yum` tasks are slow | Combine into a single task with a list of packages. |
| Serial file copies | Many `copy` or `template` tasks | Use `synchronize` module or archive + unarchive. |
| Waiting on services | `wait_for` or `uri` with retries | Increase `delay`, decrease `retries`, or use `async`. |
| Handler storms | Many handlers notified per task | Combine handlers; use `listen` to group. |

## ansible-pull

`ansible-pull` inverts the default push model. Each node pulls its own configuration from a Git repository and applies it locally. This eliminates the control node bottleneck for large fleets (1000+ nodes).

### How It Works

1. Each managed node has Ansible installed locally.
2. A cron job runs `ansible-pull` at a defined interval.
3. `ansible-pull` clones or updates a Git repository, then runs a designated playbook against `localhost`.

### Command Syntax

```bash
ansible-pull \
  -U https://github.com/your-org/ansible-config.git \
  -C main \
  -d /opt/ansible-local \
  -i localhost, \
  --full \
  site.yml
```

| Flag | Purpose |
|------|---------|
| `-U` | Repository URL to clone/pull from. |
| `-C` | Branch, tag, or commit to checkout. |
| `-d` | Directory to clone the repository into. |
| `-i localhost,` | Inventory — the trailing comma makes it a valid single-host list. |
| `--full` | Perform a full clone instead of a shallow one. |
| `site.yml` | The playbook to run after checkout. |

### Cron Job Setup

```bash
# Run ansible-pull every 30 minutes
crontab -l | {
  cat
  echo "*/30 * * * * /usr/bin/ansible-pull -U https://github.com/your-org/ansible-config.git -C main -d /opt/ansible-local -i localhost, site.yml >> /var/log/ansible-pull.log 2>&1"
} | crontab -
```

Or manage the cron job with Ansible itself (bootstrap playbook):

```yaml
- hosts: all
  tasks:
    - name: Schedule ansible-pull
      ansible.builtin.cron:
        name: "ansible-pull"
        minute: "*/30"
        job: >
          /usr/bin/ansible-pull
          -U https://github.com/your-org/ansible-config.git
          -C main
          -d /opt/ansible-local
          -i localhost,
          site.yml >> /var/log/ansible-pull.log 2>&1
```

> **Note** Use `--sleep SECONDS` to add a random delay at startup, preventing all nodes from hitting the Git server simultaneously:
>
> ```bash
> ansible-pull --sleep 60 -U https://github.com/your-org/ansible-config.git site.yml
> ```

## Mitogen

[Mitogen](https://mitogen.networkgenomics.com/ansible_detailed.html) is a third-party strategy plugin that replaces Ansible's default SSH-based task execution with a pure-Python RPC channel. It can reduce playbook execution time by 1.25x to 7x.

### Install

```bash
pip install mitogen
```

Then configure `ansible.cfg`:

```ini
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

Find the install path:

```bash
python -c "import mitogen; import os; print(os.path.dirname(mitogen.__file__))"
```

Use the parent directory of that output appended with `/ansible_mitogen/plugins/strategy`.

### Strategy Choices

| Mitogen Strategy | Equivalent Built-in |
|-----------------|-------------------|
| `mitogen_linear` | `linear` |
| `mitogen_free` | `free` |
| `mitogen_host_pinned` | `host_pinned` |

### Compatibility

- Requires Python 2.7 or Python 3.6+ on both the control node and managed nodes.
- Works with Ansible 2.10 through the version specified in Mitogen's release notes. Check the [Mitogen changelog](https://mitogen.networkgenomics.com/changelog.html) for the latest supported Ansible version.
- Compatible with most builtin modules. Some modules that manipulate the execution environment directly may not work.

### Known Limitations

- **Not compatible with all connection plugins.** Only `ssh`, `local`, and `docker` connections are supported.
- **`raw` module is not supported.** Mitogen requires Python on the remote host.
- **Async tasks behave differently.** Some async patterns may not work as expected.
- **`become` methods** are limited to `sudo` and `su`. Other become plugins (`pbrun`, `pfexec`, etc.) are not supported.
- **Binary module transfers** (e.g., compiled modules) are not supported. Only pure-Python modules work.
- **Custom connection plugins** are not supported.

> **Warning** Test Mitogen thoroughly in a staging environment before using it in production. Pin the Mitogen version in your `requirements.txt` to avoid unexpected breakage on upgrade.
