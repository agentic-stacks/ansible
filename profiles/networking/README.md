# Networking Profile

## Overview
Ansible configuration optimized for network device automation, covering switches, routers, firewalls, and load balancers across multiple vendor platforms.

## Connection Types
| Type | Use Case |
|------|----------|
| `network_cli` | CLI-based device management over SSH (most common) |
| `httpapi` | REST API-driven platforms (e.g., Arista eAPI, Fortinet) |
| `netconf` | NETCONF/YANG-capable devices (e.g., Juniper, some Cisco IOS-XE) |

Set the connection type per host or group in inventory:
```ini
[switches:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
```

## Key Collections
| Collection | Scope |
|------------|-------|
| `ansible.netcommon` | Connection plugins, common filters, base module utilities |
| `cisco.ios` | Cisco IOS / IOS-XE devices |
| `cisco.nxos` | Cisco Nexus (NX-OS) devices |
| `arista.eos` | Arista EOS switches |
| `junipernetworks.junos` | Juniper Junos routers and switches |

Install all at once:
```bash
ansible-galaxy collection install ansible.netcommon cisco.ios cisco.nxos arista.eos junipernetworks.junos
```

## Persistent Connection Settings
Network modules rely on persistent connections. Tune these in `ansible.cfg`:
```ini
[persistent_connection]
connect_timeout = 30
connect_retry_timeout = 15
command_timeout = 30
```

Increase `command_timeout` for slow devices or large config diffs.

## Applicable Skills
- `skills/foundation/configuration` — ansible.cfg tuning for network environments
- `skills/deploy/inventory` — structuring inventory by site, vendor, and device role
- `skills/deploy/modules` — using resource modules (`*_config`, `*_interfaces`, `*_vlans`)
- `skills/operations/testing` — validating network state with `assert` and `cli_parse`
