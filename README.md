# Standalone Hadoop Ansible Deployment

This project provisions a standalone, non-HA Hadoop cluster on Ubuntu Jammy
22.04 using Ansible and Vagrant with VirtualBox. The default topology contains
one NameNode and two DataNodes:

| Host | Role | Address |
| --- | --- | --- |
| `nn1` | NameNode and ResourceManager | `10.0.0.111` |
| `dn1` | DataNode and NodeManager | `10.0.0.113` |
| `dn2` | DataNode and NodeManager | `10.0.0.114` |

ZooKeeper, JournalNodes, ZKFC, NameNode failover, and ResourceManager HA are
not used. This layout is intended for local development, learning, and
standalone Hadoop testing rather than production high availability.

## Features

- Reproducible Ubuntu Jammy VMs managed by Vagrant and VirtualBox.
- Standalone HDFS and YARN configuration managed by Ansible.
- One editable VM definition in `VMs/Vagrantfile`.
- Automatic generation of `inventory/hosts.yml`.
- Automatic generation of the `hadoop_host_addresses` block in
  `inventory/group_vars/all.yml`.
- OS-aware setup for Ubuntu/Debian and Red Hat systems.
- HDFS smoke test that creates, uploads, and reads a file.

## Prerequisites

Install the following on the host running the project:

- VirtualBox
- Vagrant
- Ansible Core
- Git

The Hadoop distribution archive must also be available at:

```text
roles/hadoop/files/hadoop-3.3.6.tar.gz
```

The archive name must match `hadoop_version` in
`inventory/group_vars/all.yml`. The current default is Hadoop `3.3.6`.

## Quick Start

Clone the repository and enter it:

```bash
git clone <repository-url>
cd hadoop-ha-ansible
```

Place the Hadoop archive in `roles/hadoop/files/`, then validate and start the
VMs from the `VMs` directory:

```bash
cd VMs
vagrant validate
vagrant up
```

When Vagrant loads the configuration, it regenerates the Ansible inventory and
updates the generated address block in `inventory/group_vars/all.yml`.

Return to the repository root and verify Ansible connectivity:

```bash
cd ..
ansible all -i inventory/hosts.yml -m ping
```

Deploy Hadoop:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml
```

Run the standalone HDFS smoke test:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/smoke-test.yml
```

The deployment formats the NameNode only when its metadata directory has not
already been initialized. The Hadoop services are managed by systemd.

## Configuration

Edit the variables at the top of `VMs/Vagrantfile` to change the VirtualBox
box, VM name prefix, memory, CPU count, or node IP addresses. Add another
DataNode by adding an entry with `role: "datanode"` to `VM_SPECS`, for example:

```ruby
{ name: "dn3", role: "datanode", ip: "10.0.0.115" }
```

Then create the new VM and apply the Hadoop role to it:

```bash
cd VMs
vagrant up dn3
cd ..
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml --limit dn3
```

Do not manually edit the generated address block between these markers in
`inventory/group_vars/all.yml`:

```yaml
# BEGIN VAGRANT GENERATED HOST ADDRESSES
...
# END VAGRANT GENERATED HOST ADDRESSES
```

The remaining Hadoop settings, including `dfs_replication`, Java, service
addresses, and data directories, are maintained in
`inventory/group_vars/all.yml`.

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

Stop the VMs without deleting them:

```bash
vagrant halt
```

Delete the local VMs:

```bash
vagrant destroy -f
```

Validate the Ansible playbooks from the repository root:

```bash
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/deploy-hadoop.yml
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/smoke-test.yml
```

## Project Layout

```text
VMs/Vagrantfile                       VirtualBox VM definitions and generators
inventory/hosts.yml                   Generated Ansible inventory
inventory/group_vars/all.yml          Shared Hadoop and generated host variables
playbooks/deploy-hadoop.yml           Standalone Hadoop deployment
playbooks/smoke-test.yml              HDFS read/write verification
roles/common/                         Host preparation and Java installation
roles/hadoop/                         Hadoop installation and configuration
roles/hadoop/templates/               Hadoop XML and systemd templates
```

## Troubleshooting

- Run Vagrant commands from `VMs/`; run Ansible commands from the repository
  root.
- If an IP is changed, run `vagrant validate` or `vagrant up` before running
  Ansible so the generated inventory is refreshed.
- If Ansible reports `Permission denied (publickey)`, regenerate the inventory
  with `vagrant validate`; it contains the Vagrant private key path used for
  root access.
- Guest Additions version warnings from VirtualBox are usually non-fatal. They
  matter only if shared folders fail to mount.
- If Java installation pauses, verify that the VMs have outbound access to
  the Ubuntu package repositories.

## Support

For project-specific help, open an issue in the repository with the command
run, the relevant Ansible output, and the output of `vagrant status`. Do not
include private keys, credentials, or other secrets in issue reports.
