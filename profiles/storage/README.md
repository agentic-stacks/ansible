# Storage Profile

## Overview
Ansible configuration for managing storage infrastructure including logical volumes, network file systems, block storage, and ZFS pools across Linux environments.

## Key Technologies
| Technology | Use Case |
|------------|----------|
| LVM | Logical volume management — flexible disk partitioning and resizing |
| NFS | Network-attached shared file systems |
| iSCSI | Block-level storage over IP networks |
| ZFS | Advanced file system with snapshots, compression, and pooled storage |

## Key Modules
| Module | Purpose |
|--------|---------|
| `lvg` | Create and manage LVM volume groups |
| `lvol` | Create, resize, and remove logical volumes |
| `mount` | Mount and unmount file systems, manage `/etc/fstab` |
| `zfs` | Manage ZFS datasets (create, destroy, set properties) |
| `file` | Set ownership, permissions, and create mount-point directories |
| `filesystem` | Create file systems on block devices (ext4, xfs, etc.) |

## Key Collections
| Collection | Scope |
|------------|-------|
| `ansible.posix` | `mount`, `synchronize`, ACL management |
| `community.general` | `lvg`, `lvol`, `zfs`, `parted`, `filesystem`, and other storage utilities |

Install:
```bash
ansible-galaxy collection install ansible.posix community.general
```

## Applicable Skills
- `skills/deploy/modules` — correct usage of storage modules and idempotency patterns
- `skills/deploy/roles` — building reusable storage provisioning roles
- `skills/operations/testing` — validating mount points, volume sizes, and file system types
- `skills/operations/vault` — encrypting iSCSI CHAP credentials and NFS Kerberos keytabs
