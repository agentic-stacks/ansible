# Dynamic Inventory

## Overview

Dynamic inventory queries an external source (cloud API, CMDB, LDAP) at runtime to build the host list. Hosts are discovered automatically -- no manual file edits required. Use dynamic inventory when your infrastructure scales up and down, spans multiple cloud providers, or changes too frequently for static files.

Ansible provides two mechanisms for dynamic inventory:

1. **Inventory plugins** (recommended) -- native Python plugins configured via YAML files.
2. **Inventory scripts** (legacy) -- standalone executables that output JSON on stdout.

## How Inventory Plugins Work

1. You create a YAML configuration file with a specific filename suffix (e.g., `aws_ec2.yml`).
2. The file declares which plugin to use via the `plugin:` key.
3. Ansible calls the plugin, which queries the external API and returns hosts, groups, and variables.
4. The result merges with any other inventory sources.

## Enable Inventory Plugins in ansible.cfg

```ini
# ansible.cfg
[defaults]
inventory = ./inventory/

[inventory]
# List plugins in the order Ansible should try them
enable_plugins = amazon.aws.aws_ec2, google.cloud.gcp_compute, azure.azcollection.azure_rm, auto, host_list, yaml, ini
```

Install the required collections first:

```bash
# AWS
ansible-galaxy collection install amazon.aws

# Google Cloud
ansible-galaxy collection install google.cloud

# Azure
ansible-galaxy collection install azure.azcollection
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

## AWS EC2 Plugin

### Configuration file

The filename **must** end in `aws_ec2.yml` or `aws_ec2.yaml`.

```yaml
# inventory/prod_aws_ec2.yml
plugin: amazon.aws.aws_ec2

# AWS authentication (prefer environment variables or IAM roles in production)
aws_access_key_id: AKIAIOSFODNN7EXAMPLE
aws_secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Target specific regions
regions:
  - us-east-1
  - us-west-2

# Filter instances (only running instances with a specific tag)
filters:
  instance-state-name: running
  "tag:Environment": production

# Build groups from EC2 tags and properties
keyed_groups:
  # Group by the "Role" tag: tag_Role_webserver, tag_Role_database, etc.
  - key: tags.Role
    prefix: tag_Role
    separator: "_"

  # Group by instance type: instance_type_t3_micro, etc.
  - key: instance_type
    prefix: instance_type
    separator: "_"

  # Group by region: aws_region_us_east_1, etc.
  - key: placement.region
    prefix: aws_region
    separator: "_"

  # Group by VPC ID
  - key: vpc_id
    prefix: vpc

# Create groups conditionally
groups:
  # All instances with "web" in their Name tag
  webservers: "'web' in (tags.Name | default(''))"
  # All instances in a specific subnet
  private_subnet: "subnet_id == 'subnet-0bb1c79de3EXAMPLE'"

# Set host variables from instance attributes
compose:
  # Use private IP as the connection address
  ansible_host: private_ip_address
  # Set ansible_user based on the AMI platform
  ansible_user: "'ubuntu' if image_id.startswith('ami-0a') else 'ec2-user'"
  # Custom variable
  instance_name: tags.Name | default('unnamed')

# Use instance ID as the inventory hostname (unique and stable)
hostnames:
  - tag:Name
  - instance-id

# Include only instances matching these instance IDs (optional)
# include_instances:
#   - i-0123456789abcdef0

# Exclude specific instances (optional)
# exclude_instances:
#   - i-0fedcba9876543210
```

### Verify AWS inventory

```bash
# List all discovered hosts
ansible-inventory -i inventory/prod_aws_ec2.yml --list

# Show the group tree
ansible-inventory -i inventory/prod_aws_ec2.yml --graph

# Ping all discovered webservers
ansible -i inventory/prod_aws_ec2.yml webservers -m ping
```

### Authentication best practices

```bash
# Prefer environment variables over inline credentials
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_REGION=us-east-1

# Or use an IAM instance profile / SSO (no credentials in files)
# Just set the region:
export AWS_REGION=us-east-1
```

## Google Cloud Compute Plugin

### Configuration file

The filename **must** end in `gcp_compute.yml` or `gcp_compute.yaml`.

```yaml
# inventory/prod_gcp_compute.yml
plugin: google.cloud.gcp_compute

# Authentication
auth_kind: serviceaccount
service_account_file: /etc/ansible/gcp-sa-key.json

# Target project and zones
projects:
  - my-production-project

# Filter to specific zones (omit for all zones)
zones:
  - us-central1-a
  - us-central1-b
  - us-east1-b

# Filter instances
filters:
  - status = RUNNING
  - "labels.environment = production"

# Build groups from labels and properties
keyed_groups:
  # Group by the "role" label: label_role_webserver, etc.
  - key: labels.role
    prefix: label_role
    separator: "_"

  # Group by machine type
  - key: machineType | regex_replace('.*/', '')
    prefix: machine_type
    separator: "_"

  # Group by zone
  - key: zone | regex_replace('.*/', '')
    prefix: zone

# Set host variables
compose:
  # Use internal IP for connection
  ansible_host: networkInterfaces[0].networkIP
  # Custom variables
  instance_name: name
  gcp_zone: zone | regex_replace('.*/', '')

# Use instance name as inventory hostname
hostnames:
  - name
  - public_ip
```

### Verify GCP inventory

```bash
ansible-inventory -i inventory/prod_gcp_compute.yml --graph
ansible -i inventory/prod_gcp_compute.yml all -m ping
```

## Azure Resource Manager Plugin

### Configuration file

The filename **must** end in `azure_rm.yml` or `azure_rm.yaml`.

```yaml
# inventory/prod_azure_rm.yml
plugin: azure.azcollection.azure_rm

# Authentication (prefer environment variables in production)
auth_source: auto  # uses env vars, MSI, or CLI auth

# Target specific subscriptions (optional, defaults to all)
# subscription_id: 12345678-1234-1234-1234-123456789abc

# Include only specific resource groups
include_vm_resource_groups:
  - production-rg
  - shared-services-rg

# Exclude powered-off VMs
include_vmss_resource_groups:
  - production-vmss-rg

# Only include running VMs
conditional_groups:
  running: powerstate == "running"

# Build groups from Azure properties
keyed_groups:
  # Group by resource group: resource_group_production_rg
  - key: resource_group
    prefix: resource_group
    separator: "_"

  # Group by location: location_eastus, location_westus2
  - key: location
    prefix: location
    separator: "_"

  # Group by tags
  - key: tags.Environment | default('untagged')
    prefix: env
    separator: "_"

  # Group by OS type
  - key: os_disk.operating_system_type | default('unknown') | lower
    prefix: os
    separator: "_"

# Set host variables
compose:
  ansible_host: public_ip_address | default(private_ip_address, true)
  ansible_user: "'azureuser'"
  vm_size: virtual_machine_size

# Use VM name as inventory hostname
hostnames:
  - default  # uses the VM name

# Set default credentials via env vars:
# export AZURE_SUBSCRIPTION_ID=...
# export AZURE_CLIENT_ID=...
# export AZURE_SECRET=...
# export AZURE_TENANT=...
```

### Verify Azure inventory

```bash
ansible-inventory -i inventory/prod_azure_rm.yml --graph
ansible -i inventory/prod_azure_rm.yml all -m ping
```

## Custom Inventory Scripts (Legacy)

For sources without a native plugin, write an executable script. Ansible calls it with two arguments:

| Argument | Expected Output |
|---|---|
| `--list` | JSON dict of groups, each with `hosts` list and optional `_meta.hostvars` |
| `--host <hostname>` | JSON dict of variables for that host (can be empty `{}`) |

### Example Python script

```python
#!/usr/bin/env python3
"""Custom inventory script that queries an internal CMDB."""

import argparse
import json
import sys
import urllib.request


CMDB_URL = "https://cmdb.internal.example.com/api/v1/hosts"


def get_inventory():
    """Query CMDB and return Ansible-compatible inventory."""
    req = urllib.request.Request(CMDB_URL)
    req.add_header("Authorization", "Bearer YOUR_TOKEN_HERE")
    with urllib.request.urlopen(req) as resp:
        hosts = json.loads(resp.read())

    inventory = {
        "_meta": {
            "hostvars": {}
        }
    }

    for host in hosts:
        # Add to role-based group
        group = host["role"]
        if group not in inventory:
            inventory[group] = {"hosts": [], "vars": {}}
        inventory[group]["hosts"].append(host["fqdn"])

        # Set per-host variables
        inventory["_meta"]["hostvars"][host["fqdn"]] = {
            "ansible_host": host["ip_address"],
            "ansible_user": host.get("ssh_user", "deploy"),
            "datacenter": host.get("datacenter", "unknown"),
        }

    return inventory


def get_host(hostname):
    """Return variables for a single host."""
    inventory = get_inventory()
    return inventory["_meta"]["hostvars"].get(hostname, {})


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host", type=str)
    args = parser.parse_args()

    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### Use the script

```bash
# Make it executable
chmod +x inventory/cmdb_inventory.py

# Test it
./inventory/cmdb_inventory.py --list | python3 -m json.tool

# Use it with Ansible
ansible -i inventory/cmdb_inventory.py all -m ping
```

### Expected --list output format

```json
{
  "webservers": {
    "hosts": ["web01.example.com", "web02.example.com"],
    "vars": {
      "http_port": 80
    }
  },
  "dbservers": {
    "hosts": ["db01.example.com"],
    "vars": {
      "db_port": 5432
    }
  },
  "_meta": {
    "hostvars": {
      "web01.example.com": {
        "ansible_host": "10.0.1.11"
      },
      "web02.example.com": {
        "ansible_host": "10.0.1.12"
      },
      "db01.example.com": {
        "ansible_host": "10.0.3.31"
      }
    }
  }
}
```

The `_meta.hostvars` section is optional but recommended. When present, Ansible reads all host variables from `--list` and never calls `--host`, which is significantly faster for large inventories.

## Caching Settings for Performance

Dynamic inventory queries can be slow over the network. Enable caching to store results locally between runs.

### Configure caching in ansible.cfg

```ini
# ansible.cfg
[inventory]
enable_plugins = amazon.aws.aws_ec2, google.cloud.gcp_compute, azure.azcollection.azure_rm, auto, yaml, ini

# Enable caching
cache = true

# Cache plugin (jsonfile is simplest)
cache_plugin = ansible.builtin.jsonfile

# Where to store cache files
cache_connection = /tmp/ansible-inventory-cache

# Cache TTL in seconds (3600 = 1 hour)
cache_timeout = 3600
```

### Cache plugin options

| Plugin | Use Case | Configuration |
|---|---|---|
| `ansible.builtin.jsonfile` | Single-user, local execution | `cache_connection = /path/to/dir` |
| `ansible.builtin.redis` | Shared cache, team environments | `cache_connection = localhost:6379:0` |
| `ansible.builtin.memcached` | Shared cache, ephemeral | `cache_connection = localhost:11211` |
| `community.general.yaml` | Human-readable cache files | `cache_connection = /path/to/dir` |

### Per-plugin caching (in the inventory file)

```yaml
# inventory/prod_aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1

# Override cache settings for this specific source
cache: true
cache_plugin: ansible.builtin.jsonfile
cache_connection: /tmp/ansible-cache-aws
cache_timeout: 1800  # 30 minutes
```

### Manage the cache manually

```bash
# Force a cache refresh
ansible-inventory -i inventory/ --list --flush-cache

# Clear cache files directly
rm -rf /tmp/ansible-inventory-cache/*
```

## Combine Static and Dynamic Inventory

Use a directory-based inventory to mix static and dynamic sources:

```
inventory/
├── 01-static-hosts.yml           # manually managed hosts
├── 02-cloud-aws_ec2.yml          # auto-discovered AWS hosts
├── 03-cloud-gcp_compute.yml      # auto-discovered GCP hosts
├── group_vars/
│   ├── all.yml                   # shared variables
│   ├── tag_Role_webserver.yml    # variables for AWS-tagged webservers
│   └── label_role_database.yml   # variables for GCP-labeled databases
└── host_vars/
    └── bastion01.example.com.yml # static host overrides
```

```bash
# Ansible loads all files in the directory and merges the results
ansible-playbook -i inventory/ site.yml
```

## References

- [Working with Dynamic Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html)
- [Inventory Plugin List](https://docs.ansible.com/ansible/latest/plugins/inventory.html)
- [amazon.aws.aws_ec2 Inventory Plugin](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)
- [google.cloud.gcp_compute Inventory Plugin](https://docs.ansible.com/ansible/latest/collections/google/cloud/gcp_compute_inventory.html)
- [azure.azcollection.azure_rm Inventory Plugin](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_inventory.html)
- [Inventory Cache Plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html)
- [Developing Custom Inventory Scripts](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html)
