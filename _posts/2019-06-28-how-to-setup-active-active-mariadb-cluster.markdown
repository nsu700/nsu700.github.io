---
layout: post
title: Mariadb active-active setup
date: 2019-06-28 18:32:24.000000000 +08:00
---

## Mariadb basic installation
1. create repo for mariadb 
``` bash
[root@db1 ~]# echo '[mariadb]
> name = MariaDB
> baseurl = http://yum.mariadb.org/10.3/centos7-amd64
> gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
> gpgcheck=1'>/etc/yum.repos.d/mariadb.repo
```
2. install mariadb
``` bash
[root@db2 ~]# yum install -y MariaDB-server MariaDB-client
```

## Setup active-active
### On first node
1. create configuration for wsrep
```
wsrep_on=ON
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
query_cache_size=0
query_cache_type=0
bind-address=0.0.0.0
datadir=/var/lib/mysql
innodb_log_file_size=100M
innodb_file_per_table
innodb_flush_log_at_trx_commit=2
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://node1_IP:port1,node2_IP:port2,...[?option1=value1&...]"          # all IP of mariadb servers
wsrep_cluster_name='galera_cluster'										 
wsrep_node_address='NODE_IP'                                     
wsrep_node_name='MARIADB'                                              
wsrep_sst_method=rsync
wsrep_sst_auth=wsrep_sst_user:password    
```
    1. `wsrep_provider` defines where the galera plugin is, if this not define or is a invalid value, the node will bypass all wsrep request and act as a standard standalone instance.
    2. `wsrep_cluster_address` tells the mysql where to connect to and communicate within the cluster, never set it to "gcomm://", as it will tell the node to inital a new cluster whenever the mysql restart
    3. `wsrep_cluster_name` tells the node the cluster name, when the node joins a cluster it will first verify the name
    4. `wsrep_node_address` tells the galera plugin which address to use in the cluster, by default it will pick up the first network interface
    5. `wsrep_node_name` tells the node which name it use for logging and in cluster
    6. `wsrep_sst_method` tells the node how to request state transfer, be awear that the cluster use the same script to send and receive state transfer, so every node in the cluster should use the same setting
    7. `wsrep_sst_auth` provides the authentication info for the node when requesting state transfer
    8. `wsrep_on` defines whether replication takes place for updates from the current session.

2. Create replication user
```sql
GRANT USAGE ON *.* to sst_user@'%' IDENTIFIED BY 'dbpass';
GRANT ALL PRIVILEGES on *.* to sst_user@'%';
FLUSH PRIVILEGES;
```

3. After the configuration created, the cluster can be initilized, run the below command to bootstrap the cluster
``` bash
galera_new_cluster
```
    1. After bootstrapping, the port 4567 and 3306 should be in the listening state, then the cluster is ready

### On the other node
1. create the configuration as the first node has, but only update the `wsrep_node_address` and `wsrep_node_name`
2. start the mysql service by issuing below command
``` bash
systemctl start mysql
```

### Verify the cluster
login the database and show status like 
``` sql
SHOW STATUS LIKE 'wsrep_%';
```
for example, the `wsrep_cluster_size` should be the number of nodes in the cluster


## Reference
[Galera variables](http://galeracluster.com/library/documentation/mysql-wsrep-options.html#wsrep-cluster-address)