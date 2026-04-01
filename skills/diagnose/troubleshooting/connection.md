# Connection Troubleshooting

Decision trees and resolution steps for SSH, privilege escalation, WinRM, and network device connection failures.

## SSH Connection Decision Tree

```
Host unreachable?
├── Can you ping the host?
│   ├── No → Check: DNS resolution, IP address, host is powered on, network route
│   └── Yes
│       ├── Can you reach SSH port? (telnet/nc hostname 22)
│       │   ├── No → Firewall blocking port 22, SSH daemon not running, wrong port
│       │   └── Yes
│       │       ├── Can you SSH manually? (ssh -vvv user@hostname)
│       │       │   ├── No → See "SSH Authentication Failures" below
│       │       │   └── Yes
│       │       │       └── Ansible-specific issue → Check ansible_user, ansible_port,
│       │       │           ansible_ssh_private_key_file, ansible_connection in inventory
│       │       └──
│       └──
└──
```

## SSH Authentication Failures

### Key-Based Authentication

Debug with maximum verbosity:

```bash
ssh -vvv user@hostname
```

Common causes and fixes:

| Problem | Diagnostic | Fix |
|---|---|---|
| Wrong key offered | `ssh -vvv` shows which keys are tried | Set `ansible_ssh_private_key_file` or configure `~/.ssh/config` |
| Key permissions too open | SSH refuses key with "Permissions are too open" | `chmod 600 ~/.ssh/id_rsa` and `chmod 700 ~/.ssh/` |
| Authorized keys permissions | Remote host rejects key silently | On remote: `chmod 600 ~/.ssh/authorized_keys`, `chmod 700 ~/.ssh/`, `chmod 755 ~` |
| SSH agent not forwarding | Key exists locally but connection fails | `ssh-add -l` to verify; set `ansible_ssh_extra_args: "-o ForwardAgent=yes"` |
| Wrong user | Connecting as incorrect user | Set `ansible_user` in inventory or `-u` on command line |

### Password Authentication

```ini
# inventory
[webservers]
host1 ansible_user=deploy ansible_password=secret

# or prompt at runtime
# ansible-playbook site.yml --ask-pass (-k)
```

Install `sshpass` if using password authentication:

```bash
# Debian/Ubuntu
apt install sshpass

# RHEL/CentOS
yum install sshpass
```

## Host Key Verification

When you see `Host key verification failed`:

```bash
# View the offending key
ssh-keygen -R hostname    # remove old key
ssh-keyscan hostname >> ~/.ssh/known_hosts   # add current key
```

> **Warning:** Do not disable host key checking in production. Setting `host_key_checking = False` in `ansible.cfg` or `ANSIBLE_HOST_KEY_CHECKING=False` removes a critical security safeguard against man-in-the-middle attacks. Only use this in disposable lab or CI environments.

For initial provisioning of known-good hosts:

```ini
# ansible.cfg — scoped to a specific project, not globally
[defaults]
host_key_checking = False
```

Or per-play:

```yaml
- hosts: new_servers
  vars:
    ansible_ssh_extra_args: "-o StrictHostKeyChecking=accept-new"
```

`StrictHostKeyChecking=accept-new` is safer than `no` — it accepts keys for new hosts but still rejects changed keys.

## Privilege Escalation (become) Failures

### Decision Tree

```
"Missing sudo password" or "Incorrect sudo password"?
├── Is NOPASSWD configured for the user?
│   ├── No → Either configure NOPASSWD or use --ask-become-pass (-K)
│   └── Yes
│       ├── Is the sudoers entry correct? (visudo -c)
│       │   ├── No → Fix syntax with visudo
│       │   └── Yes
│       │       ├── Does sudo requiretty?
│       │       │   ├── Yes → Add `Defaults:ansible_user !requiretty` in sudoers
│       │       │   │   or set `become_flags: "-H -S"` in ansible.cfg
│       │       │   └── No → Check if sudo is prompting for something else (-vvvv)
│       │       └──
│       └──
└──
```

### Common Become Fixes

**NOPASSWD sudoers entry:**

```
ansible_user ALL=(ALL) NOPASSWD: ALL
```

Place in `/etc/sudoers.d/ansible` (not directly in `/etc/sudoers`) and set permissions:

```bash
chmod 440 /etc/sudoers.d/ansible
visudo -cf /etc/sudoers.d/ansible   # validate syntax
```

**Disable requiretty for Ansible user:**

```
Defaults:ansible_user !requiretty
```

**Ansible configuration:**

```yaml
# In a play
- hosts: servers
  become: true
  become_method: sudo
  become_user: root
  become_flags: "-H -S -n"   # -n = non-interactive
```

**Other become methods:**

```yaml
# su
become_method: su

# pbrun (PowerBroker)
become_method: pbrun

# doas (OpenBSD)
become_method: doas
```

## WinRM Connection Issues

### Prerequisites

Install pywinrm on the control node:

```bash
pip install pywinrm
```

### Decision Tree

```
Cannot connect to Windows host?
├── Is WinRM running on the target?
│   ├── No → On target (PowerShell admin): winrm quickconfig
│   └── Yes
│       ├── Is the correct port open? (5985 HTTP / 5986 HTTPS)
│       │   ├── No → Check Windows firewall, enable WinRM listener
│       │   └── Yes
│       │       ├── Authentication failing?
│       │       │   ├── Check ansible_winrm_transport (ntlm, kerberos, credssp, basic)
│       │       │   ├── For basic auth: must enable on target and use HTTPS
│       │       │   └── For kerberos: install pywinrm[kerberos], check krb5.conf
│       │       └── SSL certificate errors?
│       │           ├── Self-signed: ansible_winrm_server_cert_validation=ignore
│       │           └── Production: install proper CA certificate
│       └──
└──
```

### Inventory Example

```ini
[windows]
win-host1

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_scheme=https
ansible_port=5986
ansible_user=Administrator
ansible_password={{ vault_win_password }}
```

### WinRM Setup on Windows Target

```powershell
# Enable WinRM with HTTPS (recommended)
winrm quickconfig -transport:https

# Or use the Ansible setup script
# https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html
Invoke-WebRequest -Uri https://raw.githubusercontent.com/ansible/ansible-documentation/refs/heads/devel/examples/scripts/ConfigureRemotingForAnsible.ps1 -OutFile ConfigureRemotingForAnsible.ps1
powershell -ExecutionPolicy ByPass -File ConfigureRemotingForAnsible.ps1

# Verify listeners
winrm enumerate winrm/config/Listener
```

## Network Device Connections

### Connection Plugin Selection

| Device Type | Connection | Protocol |
|---|---|---|
| Most CLI-based devices (Cisco IOS, Arista EOS, Juniper) | `network_cli` | SSH |
| REST API devices (Arista eAPI, Cisco NX-API) | `httpapi` | HTTP/HTTPS |
| NETCONF-capable devices (Juniper, Cisco IOS-XE) | `netconf` | SSH (port 830) |

### Common Network Issues

```yaml
# Inventory for network devices
[routers]
router1 ansible_network_os=cisco.ios.ios

[routers:vars]
ansible_connection=network_cli
ansible_user=admin
ansible_password={{ vault_net_password }}
ansible_become=true
ansible_become_method=enable
ansible_become_password={{ vault_enable_password }}
```

**Timeouts:** Network devices may respond slowly. Increase timeouts:

```ini
# ansible.cfg
[persistent_connection]
connect_timeout = 30
command_timeout = 30
```

**SSH issues with network devices:**

```ini
# Some older devices need specific SSH settings
[ssh_connection]
ssh_args = -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa
```

**Wrong network_os:** If modules return unexpected errors, verify `ansible_network_os` matches the actual platform. Use `ansible-doc -l -t connection` to list available connection plugins.
