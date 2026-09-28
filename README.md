## Standalone Hadoop deployment

This repository deploys a standalone Hadoop cluster with one NameNode (`nn1`)
and two DataNodes (`dn1` and `dn2`).

Install Hadoop and start the services with:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml
```

Run the HDFS smoke test with:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/smoke-test.yml
```

The Hadoop archive `hadoop-3.3.6.tar.gz` must be available beside the playbook
execution context.

## VirtualBox VMs

The [VMs/Vagrantfile](VMs/Vagrantfile) creates `nn1`, `dn1`, and `dn2`. Edit
the VM and IP variables at the top of the Vagrantfile. Vagrant regenerates
`inventory/hosts.yml` and updates the generated `hadoop_host_addresses` block
in `inventory/group_vars/all.yml` whenever it loads.

The default Vagrant box is Ubuntu Jammy 22.04 (`ubuntu/jammy64`).

Edit the variables at the top of the Vagrantfile to change the box, network,
memory, CPU count, or VM definitions. Start the machines from the `VMs`
directory:

```bash
cd VMs
vagrant up
```

After the VMs are running, return to the repository root and run the Ansible
deployment:

```bash
cd ..
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml
```

Destroy the VMs when finished:

```bash
cd VMs
vagrant destroy -f
```