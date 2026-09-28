# Hadoop HA Ansible Deployment

This project provisions a Hadoop High Availability cluster with Ansible and
Vagrant using Ubuntu Jammy 22.04 and VirtualBox. The cluster uses two
NameNodes, JournalNodes, ZooKeeper, automatic NameNode failover, and YARN
ResourceManager HA.

## Topology

| Host | Services | Address |
| --- | --- | --- |
| `nn1` | NameNode, ResourceManager, JournalNode, ZooKeeper | `10.0.0.111` |
| `nn2` | NameNode, ResourceManager, ZooKeeper | `10.0.0.112` |
| `dn1` | DataNode, NodeManager, JournalNode, ZooKeeper | `10.0.0.113` |
| `dn2` | DataNode, NodeManager, JournalNode | `10.0.0.114` |
| `client` | Hadoop client and smoke tests | `10.0.0.115` |

## Features

- HDFS NameNode HA with JournalNode shared edits.
- ZooKeeper-based automatic failover through ZKFC.
- YARN ResourceManager HA.
- Ubuntu Jammy VMs managed by Vagrant and VirtualBox.
- Ansible inventory generated from shared variables.
- Ubuntu and Red Hat host preparation support.
- HA smoke test that uploads a file, fails over from `nn1` to `nn2`, and
  verifies the file remains available.

## Prerequisites

Install these tools on the host:

- VirtualBox
- Vagrant
- Ansible Core
- Git

Place the required archives in the Ansible role file directories:

```text
roles/hadoop/files/hadoop-3.3.6.tar.gz
roles/zookeeper/files/apache-zookeeper-3.8.4-bin.tar.gz
```

The archive names must match `hadoop_version` and `zk_version` in
`inventory/group_vars/all.yml`. Archives are ignored by Git and must be
provided locally.

## Quick Start

Clone the repository and enter it:

```bash
git clone <repository-url>
cd hadoop-ha-ansible
```

Review the VM definitions and shared HA variables in:

```text
VMs/Vagrantfile
inventory/group_vars/all.yml
```

`inventory/group_vars/all.yml` is the source of truth for VM addresses,
machine groups, ZooKeeper IDs, Hadoop settings, and Vagrant resources. The
Vagrantfile reads it and regenerates `inventory/hosts.yml`; it does not
overwrite `all.yml`.

Start the VMs from the `VMs` directory:

```bash
cd VMs
vagrant validate
vagrant up
```

Return to the repository root and verify connectivity:

```bash
cd ..
ansible all -i inventory/hosts.yml -m ping
```

Deploy the HA cluster:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop-ha.yml
```

Run the HA smoke test:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/smoke-test.yml
```

## Configuration

Edit `inventory/group_vars/all.yml` to change:

- VM IP addresses under `hadoop_host_addresses`.
- VM names, roles, and ZooKeeper IDs under `vagrant_vm_specs`.
- Vagrant box, memory, and CPU settings.
- Hadoop and ZooKeeper versions.
- HDFS replication and data directories.
- NameNode and ResourceManager HA addresses.

After changing shared variables, regenerate the inventory:

```bash
cd VMs
vagrant validate
```

Do not manually edit the generated `inventory/hosts.yml` file. Do not commit
`VMs/.vagrant/` or downloaded archive files.

## Useful Commands

Check VM state:

```bash
cd VMs
vagrant status
```

Connect to a VM:

```bash
vagrant ssh nn1
```

Stop the VMs:

```bash
vagrant halt
```

Destroy the VMs:

```bash
vagrant destroy -f
```

Check HA service state:

```bash
ansible namenodes -i inventory/hosts.yml -m shell \
  -a "sudo -u hadoop /opt/hadoop/bin/hdfs haadmin -getServiceState {{ inventory_hostname }}"
```

Check HDFS DataNode registration:

```bash
ansible nn1 -i inventory/hosts.yml -m shell \
  -a "sudo -u hadoop /opt/hadoop/bin/hdfs dfsadmin -report"
```

## Validation

Validate the Vagrantfile and generated inventory:

```bash
cd VMs
vagrant validate
cd ..
ansible-inventory -i inventory/hosts.yml --graph
```

Validate the Ansible playbooks:

```bash
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/deploy-hadoop-ha.yml
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/smoke-test.yml
git diff --check
```

## Project Layout

```text
VMs/Vagrantfile                       HA VirtualBox definitions and inventory generator
inventory/hosts.yml                   Generated Ansible inventory
inventory/group_vars/all.yml          Shared source-of-truth variables
playbooks/deploy-hadoop-ha.yml        HA deployment
playbooks/smoke-test.yml              HA failover and HDFS verification
roles/common/                         Host preparation and Java installation
roles/hadoop/                         Hadoop installation and HA configuration
roles/zookeeper/                      ZooKeeper installation and configuration
```

## Troubleshooting

- Run Vagrant commands from `VMs/` and Ansible commands from the repository
  root.
- If Ansible reports `Permission denied (publickey)`, run `vagrant validate`
  to regenerate the inventory with the Vagrant SSH key.
- If Hadoop reports zero DataNodes, check that `getent hosts nn1 dn1 dn2`
  returns the `10.0.0.x` addresses and that the DataNode services are active.
- If Java installation pauses, verify outbound access from the VMs to the
  Ubuntu package repositories.
- Guest Additions mismatch warnings are generally non-fatal unless shared
  folders fail to mount.

## Support and Contributions

Open a repository issue with the command run, relevant output, and
`vagrant status`. Do not include private keys or credentials. Contributions
should preserve the HA topology, keep `all.yml` as the source of truth, and
include syntax or runtime validation for deployment changes.

## License

No `LICENSE` file is currently included in this repository.