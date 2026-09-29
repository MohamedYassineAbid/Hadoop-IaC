# Hadoop Standalone Cluster with Ansible

This repository provisions a small Hadoop and Hive laboratory cluster on Ubuntu VMs using Vagrant, VirtualBox, and Ansible.

## Architecture

| Host | IP | Services |
| --- | --- | --- |
| `nn1` | `10.0.0.111` | NameNode, ResourceManager, Hive Metastore, HiveServer2, MySQL |
| `dn1` | `10.0.0.113` | DataNode, NodeManager |
| `dn2` | `10.0.0.114` | DataNode, NodeManager |

| Component | Version or path |
| --- | --- |
| Ubuntu guest | 22.04 Jammy |
| Hadoop | `3.3.6`, `/opt/hadoop` |
| Hive | `3.1.3`, `/usr/local/hive` |
| Hadoop runtime | Java 11 |
| Hive runtime | Java 8 (`/usr/lib/jvm/java-8-openjdk-amd64`) |
| HDFS default filesystem | `hdfs://nn1:8020` |
| HDFS replication | `2` |
| Hive metastore database | MySQL database `hive_metastore` |

## Prerequisites

Install these on the Ansible controller:

- VirtualBox
- Vagrant
- Ansible Core
- Git

Put the distribution archives in the role file directories:

```text
roles/hadoop/files/hadoop-3.3.6.tar.gz
roles/hive/files/apache-hive-3.1.3-bin.tar.gz
```

The archive names must match `hadoop_version` and `hive_version` in `inventory/group_vars/all.yml`.

## For MAC/Linux Users

The latest version of Virtualbox for Mac/Linux can cause issues.

Create/edit the /etc/vbox/networks.conf file and add the following to avoid any network-related issues.
<pre>* 0.0.0.0/0 ::/0</pre>

or run below commands

```shell
sudo mkdir -p /etc/vbox/
echo "* 0.0.0.0/0 ::/0" | sudo tee -a /etc/vbox/networks.conf
```
## First Deployment

Run Vagrant commands from `VMs/` and Ansible commands from the repository root:

```bash
cd VMs
vagrant validate
vagrant up
cd ..
ansible all -i inventory/hosts.yml -m ping
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml
```

`Vagrantfile` regenerates `inventory/hosts.yml` and the generated host address block in `inventory/group_vars/all.yml`. Do not manually edit the block between:

```yaml
# BEGIN VAGRANT GENERATED HOST ADDRESSES
# END VAGRANT GENERATED HOST ADDRESSES
```

The Hadoop deployment jobs are:

1. Prepare every host, including `/etc/hosts`, firewall settings, Java 11, and the Hadoop shell environment.
2. Extract Hadoop and create `/opt/hadoop`.
3. Generate Hadoop XML configuration and `hadoop-env.sh`.
4. Install systemd units for NameNode, DataNode, ResourceManager, and NodeManager.
5. Format the NameNode only when its metadata directory is not initialized.
6. Start Hadoop services on their correct hosts.

## Hive Deployment

Hive is enabled by default:

```yaml
hive_enabled: true
```

Install or repair Hive independently with:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install-hive.yml
```

The Hive playbook runs these jobs on `nn1`:

1. Apply the common host and Hadoop CLI environment.
2. Install MySQL server and client.
3. Install Java 8 for Hive 3.1.3.
4. Start MySQL.
5. Extract Hive and create `/usr/local/hive`.
6. Download the MySQL Connector/J driver.
7. Generate `hive-env.sh` and `hive-site.xml`.
8. Create the MySQL database and `hive` user.
9. Create HDFS warehouse and temporary directories.
10. Initialize the Hive metastore schema.
11. Install and start Hive Metastore and HiveServer2.

Set `hive_enabled: false` to skip Hive during the full deployment.

## Services and Ports

### Hadoop services

| Service | Host | Port | Purpose |
| --- | --- | ---: | --- |
| NameNode RPC | `nn1` | `8020` | HDFS client and DataNode RPC |
| NameNode web UI | `nn1` | `9870` | HDFS administration and status |
| ResourceManager client | `nn1` | `8032` | YARN client communication |
| ResourceManager scheduler | `nn1` | `8030` | Application scheduling |
| ResourceManager tracker | `nn1` | `8031` | NodeManager resource tracking |
| ResourceManager admin | `nn1` | `8033` | YARN administration |
| ResourceManager web UI | `nn1` | `8088` | YARN applications and cluster status |
| DataNode data transfer | `dn1`, `dn2` | `9866` | HDFS block reads and writes |
| DataNode web UI | `dn1`, `dn2` | `9864` | DataNode status and blocks |
| DataNode IPC | `dn1`, `dn2` | `9867` | DataNode internal RPC |
| NodeManager web UI | `dn1`, `dn2` | `8042` | NodeManager status and containers |
| MapReduce shuffle | `dn1`, `dn2` | `13562` | MapReduce intermediate data shuffle |

DataNode, NodeManager, and shuffle ports use Hadoop defaults because this repository does not override them.

### Hive and database services

| Service | Host | Port | Purpose |
| --- | --- | ---: | --- |
| Hive Metastore | `nn1` | `9083` | Thrift metadata service |
| HiveServer2 | `nn1` | `10000` | Beeline and JDBC/Thrift clients |
| MySQL | `nn1` | `3306` | Hive metastore database; normally loopback only |

There is no ZooKeeper, JournalNode, ZKFC, or HA service in this topology.

Check listeners on a guest:

```bash
ss -ltnp
systemctl --type=service --state=running | grep -E 'hadoop|hive|mysql'
```

## Systemd Service Names

| Unit | Expected host |
| --- | --- |
| `hadoop-namenode.service` | `nn1` |
| `hadoop-resourcemanager.service` | `nn1` |
| `hadoop-datanode.service` | `dn1`, `dn2` |
| `hadoop-nodemanager.service` | `dn1`, `dn2` |
| `mysql.service` | `nn1` |
| `hive-metastore.service` | `nn1` |
| `hive-server2.service` | `nn1` |

Useful service commands:

```bash
sudo systemctl status hadoop-namenode
sudo systemctl status hive-metastore hive-server2 mysql
sudo systemctl restart hive-metastore hive-server2
sudo journalctl -u hive-server2 -n 100 --no-pager
```

## Hadoop CLI Usage

The common role installs `/etc/profile.d/hadoop.sh`. A new login shell has `/opt/hadoop/bin` and `/opt/hadoop/sbin` in `PATH`, plus the `hdfs` command:

```bash
vagrant ssh nn1
su - hadoop

hdfs dfs -mkdir -p /test/input
echo 'hello hdfs' >/tmp/test.txt
hdfs dfs -put -f /tmp/test.txt /test/input/
hdfs dfs -ls /test/input
hdfs dfs -cat /test/input/test.txt
```

For convenience, the login profile also defines `dfs` as a function forwarding to `hdfs dfs`:

```bash
dfs -ls /
dfs -cat /test/input/test.txt
```

For an existing shell, start a new login shell or run:

```bash
source /etc/profile.d/hadoop.sh
```

Useful HDFS administration commands:

```bash
hdfs dfsadmin -report
hdfs fsck / -files -blocks
hdfs dfs -df -h /
hdfs dfs -du -h /
```

## YARN Usage

```bash
yarn node -list
yarn application -list
yarn queue -status default
```

Web UIs:

```text
NameNode:       http://10.0.0.111:9870
ResourceManager: http://10.0.0.111:8088
```

## Hive Usage

Connect to HiveServer2 with Beeline:

```bash
/usr/local/hive/bin/beeline \
  -u 'jdbc:hive2://localhost:10000/default' \
  -n hadoop
```

Run a one-shot query:

```bash
/usr/local/hive/bin/beeline \
  -u 'jdbc:hive2://localhost:10000/default' \
  -n hadoop \
  -e 'SHOW DATABASES;'
```

Create and query a test table:

```bash
/usr/local/hive/bin/beeline \
  -u 'jdbc:hive2://localhost:10000/default' \
  -n hadoop \
  -e "CREATE TABLE IF NOT EXISTS demo (id INT, name STRING); \
      INSERT INTO demo VALUES (1, 'one'); \
      SELECT * FROM demo;"
```

Hive uses MapReduce by default through `hive_execution_engine: mr`. This development configuration disables HiveServer2 `doAs` impersonation and metastore notification API authorization. Review those security settings before production use.

## Tests and Health Checks

### Ansible syntax checks

```bash
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/deploy-hadoop.yml
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/install-hive.yml
ansible-playbook --syntax-check -i inventory/hosts.yml playbooks/smoke-test.yml
```

### Connectivity

```bash
ansible all -i inventory/hosts.yml -m ping
```

### HDFS smoke test

This creates a local file, uploads it to HDFS, and reads it back:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/smoke-test.yml
```

Expected output includes:

```text
File content: HA test
```

### Hive functional test

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install-hive.yml
ansible nn1 -i inventory/hosts.yml -m ansible.builtin.wait_for \
  -a 'port=10000 host=127.0.0.1 timeout=90'
ansible nn1 -i inventory/hosts.yml -m shell -a \
  "/usr/local/hive/bin/beeline -u 'jdbc:hive2://localhost:10000/default' \
   -n hadoop -e 'SHOW DATABASES;'"
```

Expected output includes `Connected to: Apache Hive (version 3.1.3)` and the `default` database.

### Direct service checks

```bash
ansible nn1 -i inventory/hosts.yml -m shell -a \
  'systemctl is-active mysql hive-metastore hive-server2'
ansible nn1 -i inventory/hosts.yml -m shell -a \
  "ss -ltn | grep -E ':(3306|9083|10000)\\b'"
ansible nn1 -i inventory/hosts.yml -m shell -a \
  '/usr/local/hive/bin/schematool -dbType mysql -info'
```

## Configuration Reference

Edit `inventory/group_vars/all.yml` for the main settings:

| Variable | Meaning |
| --- | --- |
| `hadoop_version` | Hadoop archive version |
| `hadoop_home` | Hadoop installation path |
| `java_home` | Java used by Hadoop |
| `fs_default` | HDFS URI, normally `hdfs://nn1:8020` |
| `dfs_replication` | Number of HDFS block replicas |
| `dfs_namenode_name_dir` | NameNode metadata directory |
| `dfs_datanode_data_dir` | DataNode block directory |
| `hive_enabled` | Enable or skip Hive |
| `hive_version` | Hive archive version |
| `hive_java_home` | Java used by Hive services |
| `hive_metastore_port` | Hive Metastore Thrift port |
| `hive_server2_port` | HiveServer2 Thrift port |
| `hive_metastore_db` | MySQL database name |
| `hive_metastore_user` | MySQL user |
| `hive_metastore_password` | MySQL password |
| `hive_execution_engine` | Hive execution engine, default `mr` |

Change secrets before using this outside a disposable lab. The default metastore password is intended only for local testing.

## Adding a DataNode

Add a DataNode entry to `VMs/Vagrantfile`:

```ruby
{ name: "dn3", role: "datanode", ip: "10.0.0.115" }
```

Create it and apply Hadoop:

```bash
cd VMs
vagrant validate
vagrant up dn3
cd ..
ansible-playbook -i inventory/hosts.yml playbooks/deploy-hadoop.yml --limit dn3
ansible nn1 -i inventory/hosts.yml -m shell -a 'hdfs dfsadmin -report'
```

## Vagrant Operations

```bash
cd VMs
vagrant status
vagrant ssh nn1
vagrant halt
vagrant destroy -f
```

Use `vagrant validate` after changing VM definitions. Vagrant updates the generated inventory and host address variables when it evaluates the file.

## Troubleshooting

### `hdfs` or `dfs` is not found

```bash
source /etc/profile.d/hadoop.sh
command -v hdfs
type dfs
```

Expected paths include `/opt/hadoop/bin/hdfs` and a `dfs` shell function.

### Ansible cannot connect

```bash
cd VMs
vagrant validate
vagrant status
cd ..
ansible all -i inventory/hosts.yml -m ping
```

If SSH authentication fails, regenerate the inventory with Vagrant. It contains the Vagrant private key path used for root access.

### HiveServer2 is active but port `10000` is closed

```bash
systemctl status hive-server2 --no-pager
journalctl -u hive-server2 -n 100 --no-pager
ss -ltnp | grep -E ':(9083|10000)\\b'
sudo systemctl restart hive-metastore
sudo systemctl restart hive-server2
```

### Hive metastore schema problems

```bash
/usr/local/hive/bin/schematool -dbType mysql -info
mysql --protocol=socket -uroot -e 'SHOW DATABASES;'
```

Do not initialize a populated database repeatedly. The role uses `/usr/local/hive/.metastore-schema-initialized` as its local marker.

### Java problems

Hadoop uses Java 11 and Hive uses Java 8:

```bash
grep JAVA_HOME /opt/hadoop/etc/hadoop/hadoop-env.sh
systemctl show hive-server2 -p Environment --value
ps -ef | grep -E '[h]ive-(service|metastore)'
```

Hive `3.1.3` has Java 11 classloader incompatibilities, so test HiveServer2 and Beeline before changing the Hive Java setting.

## Repository Layout

```text
VMs/Vagrantfile                         VM definitions and inventory generation
inventory/hosts.yml                     Generated Ansible inventory
inventory/group_vars/all.yml            Cluster, Hadoop, and Hive variables
playbooks/deploy-hadoop.yml             Full Hadoop deployment and service start
playbooks/install-hive.yml              Common environment plus Hive deployment
playbooks/smoke-test.yml                HDFS create, upload, and read test
roles/common/                           Hosts, Java 11, and Hadoop shell profile
roles/hadoop/                            Hadoop installation and configuration
roles/hive/                              MySQL, Hive, metastore, and HiveServer2
```

This is a development and learning cluster. Review security, passwords, firewalls, storage, HA architecture, and service users before adapting it for production.
