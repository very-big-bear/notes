## 基本說明
1. 主要紀錄本次架設時進行的設定操作，其餘較詳細的部分可以參考先前筆記
2. 


---
## SLES15_install
0. 較為詳細的安裝過程請參考>>>[02. SUSE 15 Install.md](https://github.com/HongScarlet/homework/blob/master/SUSE15%20cluster/02.%20SUSE%2015%20Install.md) 
1. bios 關掉 Hyper Threading

2. 硬碟分割參考
```bash
# xeon
sda
├─sda1  512MB  /boot
├─sda2  192GB  [SWAP]
├─sda3   30GB  /
├─sda4     1K  
├─sda5   30GB  /usr
└─sda6  other  /tmp
```
```bash
# gpu (做lvm)
sda
├─sda1         512MB  /boot
└─sda2
  ├─vg00-lv00    2GB  [SWAP]
  └─vg00-lv01  256GB  /
```

---
## Firewall
```bash
#slurm
linux :~# firewall-cmd --list-all
linux :~# firewall-cmd --add-port=6817/tcp --add-port=6818/tcp --permanent
linux :~# firewall-cmd --add-port=60001-65000/tcp --permanent
linux :~# firewall-cmd --reload
linux :~# firewall-cmd --list-all
#nis
linux :~# firewall-cmd --add-port=1011/upd --permanent
用yast打開firewall (rpc-bind)
#ssh
用yast打開firewall (ssh)
#nfs
用yast打開firewall (nfs)
```
---
## ssh setting
```bash
#開啟服務
linux :~# systemctl enable sshd.service
#免密碼登入設置
linux :~# ssh-keygen
linux :~# ssh-copy-id root@<id>
#登入
linux :~# ssh <id or name>
linux :~# logout #(登出)
#複製
linux :~# scp <資料位置> <id or name>:<放置位置>
```
---
## munge
```bash
#munge install (server, client皆要裝)
linux :~# zypper munge
linux :~# systemctl enable munge.service
linux :~# systemctl start munge.service
#munge
linux :~# cd /etc/munge/
linux :~# md5sum munge.key
linux :~# scp /etc/munge/munge.key <client_name>:/etc/munge/. #server, client一致
#munge test
linux :~# munge -n
linux :~# munge -n | unmunge
linux :~# munge -n | ssh <id or name> unmunge
```
---
## chrony
```bash
#service
sle :~# systemctl enable chronyd
sle :~# systemctl start chronyd
#config
sle :~# vi /etc/chrony.conf #server <server_ip>
sle :~# ls /etc/chrony.d
#command
sle :~# chronyc -a makestep
sle :~# chronyc -a 'burst 4/4'
```
---
## nis
```bash
#open passwd, group, hosts, shadow
#config
linux :~# vi /etc/ypserv.conf
#service
linux :~# systemctl ypserv.servic
```
---
## nfs
```bash
#rw, async, no_root_squash
```
---
## slurm install and setting
```bash
#install
linux :~# zypper in slurm
for server
linux :~# zypper in slurm-node
for server
```

```bash
#open system
linux :~# systemctl enable slurmctld.service
linux :~# systemctl start slurmctld.service (for server)
linux :~# systemctl enable slurmd.service
linux :~# systemctl start slurmd.service (for client)
```

```bash
#config setting
linux :~# vi /etc/slurm/slurm.conf
ClusterName=<自訂統一名稱>
SlurmctldHost=<server_name>(<server_id>)
SlurmUser=root
SlurmdUser=root
SlurmctldPort=6817
SlurmPort=6818
SrunPorRange=60001-65000
NodeName=<client_name>
State=idle
CPUs=<cpu_number>
PartitionName=normal
Nodes=<client_name>(ex:c[1-8])
Default=YES
```

## QOS
```bash
# List cluster
server :~ # sacctmgr list cluster
server :~ # sacctmgr list cluster format=Cluster, QOS
# Add cluster
server :~ # sacctmgr add cluster linux               #要與slurm.conf 中的ClusterName 相同
server :~ # sacctmgr list cluster format=Cluster, QOS
```
```bash
# Add Account/User
server :~ # sacctmgr add account jjcusers cluster=linux
server :~ # sacctmgr add user name=test1 account=jjcuser cluster=linux

server :~ # sacctmgr list assoc format=Cluster,Account,User,QOS
  Cluster   Account     User      QOS
 ```````` ````````` ```````` `````````
    linux      root             normal
    linux      root     root    normal
    linux  jjcusers             normal
    linux  jjcusers    test1    normal
    
```

```bash
# Add QOS
server :~ sacctmgr add qos jjcqos1
server :~ sacctmgr modify qos jjcqos1 set priority=10     #數字越小越慢開始
server :~ sacctmgr modify qos jjcqos1 set GrpJobs=30      #整個Group的Job數量
server :~ sacctmgr modify qos jjcqos1 set MaxJobsPU=3     #個人可以算的Job數量
server :~ sacctmgr show qos format=name,priority,GrpJobs
    Name      Priorty     GrpJobs     MaxJobsPU
 ``````` ```````````` ```````````  ````````````
  normal            0          30               
     low           10          30             3
 jjcqos1          100          30             4
     
# Modify user's QOS
server :~ # sacctmgr modify user test1 set qos=jjcqos1
server :~ # sacctmgr show assoc format=cluster,user,qos
  Cluster   Account     User      QOS
 ```````` ````````` ```````` `````````
    linux      root             normal
    linux      root     root    normal
    linux  jjcusers             normal
    linux  jjcusers    test1   jjcqos1
```
## MariaDB

### package

```bash
controller:~ # zypper in mariadb

# setup mariadb root
controller:~ # /usr/bin/mysqladmin -u root password <new-password>
controller:~ # /usr/bin/mysqladmin -u root -h <hostname> password <new-password>
controller:~ # /usr/bin/mysql_secure_installation
```


### config

```bash
controller:~ # vi /etc/my.cnf
# increase pool size
innodb_buffer_pool_size = 128M
```


### daemon

```bash
controller:~ # systemctl start mariadb.service
controller:~ # systemctl enable mariadb.service
```


### check

```bash
controller:~ # mysql -u root
-- check pool size
MariaDB> show variables like 'innodb_buffer_pool_size';

-- check db engine
MariaDB> show engines;

MariaDB> quit;
```

### db

```sql
-- create db
MariaDB> create database slurm_acct_db;
MariaDB> show databases;
MariaDB> drop database slurm_acct_db;
```


### user

```sql
-- create user
MariaDB> create user 'slurm'@'localhost' identified by '<password>';
MariaDB> create user 'slurm'@'controller' identified by '<password>';
MariaDB> select Host, User, Password from mysql.user;
MariaDB> drop user 'slurm'@'localhost';
MariaDB> drop user 'slurm'@'controller';

-- change password
MariaDB> set PASSWORD FOR 'slurm'@'localhost' = PASSWORD('<password>');
MariaDB> set PASSWORD FOR 'slurm'@'controller' = PASSWORD('<password>');


-- setup grant privilege
MariaDB> grant all on slurm_acct_db.* TO 'slurm'@'localhost';
MariaDB> grant all on slurm_acct_db.* TO 'slurm'@'controller';
MariaDB> show grants for slurm@localhost;
```

### test

```bash
controller:~ # mysql -u slurm
MariaDB> use slurm_acct_db;
```


---

## SlurmDB Daemon

```
         +-------------------+
         controller     compute node
         192.168.0.1    192.168.0.101
service: munge          munge
         ypserv         ypbind
         slurmctld      slurmd
         mariadb
         slurmddb
```

### package

```bash
controller:~ # zypper in slurm-slurmdbd
```


### slurm config

```bash
controller:~ # vi /etc/slurm/slurm.conf
AccountingStorageHost=controller
AccountingStorageUser=slurm
AccountingStoragePass=/var/run/munge/munge.socket.2
AccountingStoragePort=6819
AccountingStorageType=accounting_storage/slurmdbd

JobAcctGatherType=jobacct_gather/cgroup

JobCompType=jobcomp/none
```

AccountingStorageType: accounting_storage/none, accounting_storage/filetxt, accounting_storage/slurmdbd

JobAcctGatherType: jobacct_gather/none, jobacct_gather/linux, jobacct_gather/cgroup

JobCompType: jobcomp/none, jobcomp/elasticsearch, jobcomp/filetxt, jobcomp/mysql, jobcomp/script

[slurm.conf](https://slurm.schedmd.com/slurm.conf.html)


### slurmdbd config

```bash
controller:~ # vi /etc/slurm/slurmdbd.conf
StorageType=accounting_storage/mysql

StorageUser=slurm
StoragePass=<password>
StorageLoc=slurm_acct_db
```

[slurmdbd.conf](https://slurm.schedmd.com/slurmdbd.conf.html)


### daemon

```bash
controller:~ # systemctl restart slurmctld.service

controller:~ # systemctl start slurmdbd.service
controller:~ # systemctl enable slurmdbd.service
```


### check db

```bash
controller:~ # mysql -u root
MariaDB> show databases;
MariaDB> use slurm_acct_db;
MariaDB> show tables;
```

```bash
controller: # sacct
```
