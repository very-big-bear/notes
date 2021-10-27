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
