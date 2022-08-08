## 基本說明
1. 主要紀錄本次架設時進行的設定操作


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
## 內容
```bash
1. 安裝 SUSE sle
2. 設定 ssh, munge, nis, nfs, chrony
3. 編譯 compiler, vasp
4. 設定 slurm
5. 設定 MairaDB, SlurmDB (only server)
6. 設定 QOS (only server)

```
## Firewall
```bash
#slurm
linux :~# firewall-cmd --list-all
linux :~# firewall-cmd --add-port=6817/tcp --add-port=6818/tcp --permanent
linux :~# firewall-cmd --add-port=6819/tcp --permanent  #for slurmdb
linux :~# firewall-cmd --add-port=60001-65000/tcp --permanent

#nis
linux :~# firewall-cmd --add-port=1011/tcp --add-port=1011/udp --permanent

#ssh
用yast打開firewall (ssh)

#nfs
用yast打開firewall (nfs,rpc-bind,mountd)
linux :~# firewall-cmd --add-port=111/tcp --add-port=111/udp--permanent
linux :~# firewall-cmd --add-port=2049/tcp --add-port=2049/udp--permanent
linux :~# firewall-cmd --add-port=4000-4004/tcp --add-port=4000-4004/udp--permanent
linux :~# firewall-cmd --reload
linux :~# firewall-cmd --list-all

#ntp
用yast打開firewall (ntp)

#MySQL, MariaDB
linux :~# firewall-cmd --add-port=3306/tcp --add-port=3306/udp--permanent

#VPN
linux :~# firewall-cmd --add-port=1723/tcp --add-port=1723/udp--permanent

#reload, check
linux :~# firewall-cmd --reload
linux :~# firewall-cmd --list-all

#問題
nis,nfs等都不通，先關server防火牆，確認是否為防火牆問題
大多數不通都是防火牆在搞
如果要通kvm，在libvirt打開
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
linux :~# logout # =ctrl+D (登出)
#複製
linux :~# scp <資料位置> <id or name>:<放置位置> #key的位置/root/.ssh/
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
linux :~# scp /etc/munge/munge.key <client_name>:/etc/munge/. #server, client要一致
#munge test
linux :~# munge -n
linux :~# munge -n | unmunge
linux :~# munge -n | ssh <id or name> unmunge
#問提
munged: Error: Keyfile is insecure: /etc/munge/munge.ky should be owned UID 461 insted of UID 0
解決方法；修改munge.ky 權限 (可能不小心動到，導致權限鎖住)
linux :~# chown munge: munge.key
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

##ntp
```bash
#server
now and on boot
Dynamic
#client
now and on boot
Static
<server or ip>
```

---
## nis
```bash
# open passwd, group, hosts, shadow
# 255.255.255.0 192.168.1.0
#config
linux :~# vi /etc/ypserv.conf
# service
linux :~# systemctl ypserv.servic
# test
getent passwd
getent group
...
```
---
## nfs
```bash
#ex: 192.168.1.0/24 
#rw, async, no_root_squash, no_subtree_check
linux :~# vi /etc/sysconfig/nfs
 LOCKD_TCPPORT=40001
 LOCKD_UDPPORT=40001
 MOUNTD_PORT=40002
 STATD_PORT=40003
```
---
## slurm install and setting
```bash
#install
linux :~# zypper in slurm
for server
linux :~# zypper in slurm-node
for client
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
```bash
# slurm 問題
/var/log/slurm.ctld.log #log 檔位置
controller:~ # rm /var/lib/slurm/clustername #可以改clustername
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

### use

```bash
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
server :~ sacctmgr show qos format=name,priority,GrpJobs,MaxJobsPU
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
## QoS

### scontrol

```bash
controller:~ # scontrol show config
controller:~ # scontrol show config | grep SchedulerType
controller:~ # scontrol show config | grep PriorityType
controller:~ # scontrol show config | grep AccountingStorageEnforce
controller:~ # scontrol show config | grep PriorityWeightQOS
```

SchedulerType: sched/wiki -> maui, sched/wiki2 -> moab, sched/builtin or sched/backfill -> slurm

PriorityType: priority/basic -> fifo, priority/multifactor -> job priority factor

AccountingStorageEnforce: limits

PriorityWeightQOS: =0 don't use the qos factor, != 0 use the qos factor


### sacctmgr

```bash
controller:~ # sacctmgr list qos [format=Name,Priority,GrpCPUs]
controller:~ # sacctmgr add qos <qos> [Priority=1000]
controller:~ # sacctmgr del qos <qos>
controller:~ # sacctmgr mod qos <qos> set GrpCPUs=-1 Flags=OverPartQOS # -1 is default, unlimited
controller:~ # sacctmgr mod qos <qos> set GrpJobs=<n>   # job number
controller:~ # sacctmgr mod qos <qos> set Priority=<n>  # job priority

# associate qos
controller:~ # sacctmgr mod account <account> set qos=<qos>
controller:~ # sacctmgr mod user <user> set qos=<qos>

controller:~ # sacctmgr list associations
```

[Resource Limits](https://slurm.schedmd.com/resource_limits.html)

### sprio

```bash
controller:~ # sprio
controller:~ # sprio -l
```


### example

```bash
# for qos
controller:~ # sacctmgr list configuration
controller:~ # sacctmgr list cluster
controller:~ # sacctmgr list qos format=name,priority,usagefactor
controller:~ # sacctmgr list qos format=name,maxsubmitjobsperuser,maxjob
controller:~ # sacctmgr list qos format=name,grpsubmitjob,grpjob
controller:~ # sacctmgr list account
controller:~ # sacctmgr list user
controller:~ # sacctmgr list association format=qos,account,user
controller:~ # sacctmgr list stats

controller:~ # sacctmgr add qos high_qos   priority=1000 usagefactor=1.0
controller:~ # sacctmgr add qos medium_qos priority=100  usagefactor=0.8
controller:~ # sacctmgr add qos low_qos    priority=10   usagefactor=0.5

controller:~ # sacctmgr add account high_acc   cluster=mycluster qos=high_qos
controller:~ # sacctmgr add account medium_acc cluster=mycluster qos=medium_qos
controller:~ # sacctmgr add account low_acc    cluster=mycluster qos=low_qos

controller:~ # sacctmgr add user name=high_user   account=high_acc   cluster=mycluster
controller:~ # sacctmgr add user name=medium_user account=medium_acc cluster=mycluster
controller:~ # sacctmgr add user name=low_user    account=low_acc    cluster=mycluster

# for job
controller:~ # squeue
controller:~ # squeue -l

controller:~ # sinfo
controller:~ # sinfo -al
controller:~ # sinfo -Nal
controller:~ # sinfo -N -o "%.20N %.15C %.10t %.10m %.15P %.15G %.35E"

controller:~ # sprio
controller:~ # sprio -nl
controller:~ # sprio -nl

controller:~ # sshare
controller:~ # sshare -al

controller:~ # sstat -j <job_id>

# config
controller:~ # scontrol show node
controller:~ # scontrol show partition
controller:~ # scontrol show job

# repot
controller:~ # sreport cluster UserUtilizationByAccount
controller:~ # sreport user TopUsage

# 管理cluster
controller:~ # scontrol update nodename=<nodename> State=DRAIN Reason="隨便打"
controller:~ # scontrol update nodename=<nodename> State=UP
# 批量砍job
controller:~ # scancel {jobid..jobid}
```
### 學長們筆記
```
https://github.com/wsunccake/myNote/tree/master/os/linux/sle15
https://github.com/HongScarlet/homework/blob/master/SUSE15%20cluster/15.%20SLES%2015%20Cluster%20New.md#munge
```
