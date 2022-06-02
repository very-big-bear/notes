## 目錄
* [目錄](#目錄)
* [基本說明](#基本說明)
* [SLES15_install](#SLES15_install)
* [Network_Setting_Security](#Network_Setting_Security)
   * [basic](#basic)
   * [fail2ban](#fail2ban)
   * [sshd](#sshd)
   * [visudo](#visudo)
* [NFS](#NFS)
* [NIS](#NIS)
* [chrony](#chrony)
* [munge](#munge)
* [slurm](#slurm)
   * [install](#install)
   * [port](#port)
   * [slurm_conf](#slurm_conf)
   * [QOS](#QOS)
   * [Problem](#Problem)
* [intel_compiler](#intel_compiler)
* [nvidia](#nvidia)
* [vasp](#vasp)
   * [vasp_std](#vasp_std)
   * [vasp_gpu](#vasp_gpu)
   * [Problem](#Problem)
---


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
## Network_Setting_Security
#### basic
```bash
linux :~ # yast
System > Network Settings
1. Static IP(對內) or DHCP(對外)
2. hostname and dns(對外的機器使用dhcp，請把Set Hostname via DHCP關掉)
3. MTU = 9000

linux :~ # ip a
```

#### fail2ban
`install`
```bash
linux :~ # zypper addrepo https://download.opensuse.org/repositories/network:utilities/SLE_15_SP2/network:utilities.repo
linux :~ # zypper refresh
linux :~ # zypper install python-pyinotify

linux :~ # zypper addrepo https://download.opensuse.org/repositories/security/SLE_15_SP2/security.repo
linux :~ # zypper refresh
linux :~ # zypper install fail2ban

linux :~ # systemctl enable fail2ban --now
```

`setting`
```bash
linux :~ # vi /etc/fail2ban/jail.conf

# 白名單
[DEFULT]
ignoreip = 127.0.0.1/32 192.168.1.0/24

# findtime 內登入失敗 maxtry 次，則封鎖bantime
bantime  = 10m
findtime  = 10m
maxretry = 3

[sshd]
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
ignoreip = 192.168.1.0/24

● 各項目有設定時，會覆蓋DEFULT內的設定
```

`command`
```bash
# fail2ban-client
linux :~ # fail2ban-client status
linux :~ # fail2ban-client status <jail>

linux :~ # fail2ban-client add <jail>
linux :~ # fail2ban-client create <jail>
linux :~ # fail2ban-client stop <jail>
set <JAIL> addignoreip <IP>
```
command參考 >>>[fail2ban wiki](https://www.fail2ban.org/wiki/index.php/Commands) 

#### sshd
`admin`
```bash
linux :~ # vi /etc/ssh/sshd_config
PermitRootLogin no    # 禁止root登入
AllowUsers <usrename> # 允許特定user登入

linux :~ # systemctl restart sshd
```

`login`
```bash
linux :~ # vi /etc/ssh/sshd_config

# 禁用密碼登入.驗證
PasswordAuthentication no
ChallengeResponseAuthentication no

linux :~ # systemctl restart sshd
```

```bash
linux :~ # ssh-copy-id root@<client>
linux :~ # ssh-keygen -t rsa -P '' -f /home/<user>/.ssh/id_rsa   # dsa有可能安全性因素而無法登入
```

#### visudo
```bash
linux :~ # visudo

User_Alias      ADMINS = <username>
ADMINS ALL=(ALL) ALL
```
---
## NFS
server
```bash
server :~ # yast
Network Services > NFS Server
# Enter NFSv4 domain name

Host Wild Card  Options
192.168.1.0/24  rw,no_root_squash,async,no_subtree_check
client :~ # mkdir work1 work2
client :~ # yast
```

client
```bash
# 掛載 work1 work2 home
client :~ # mkdir work1 work2
client :~ # yast
Network Services > NFS Client

# 掛載repo
client :~ # zypper rr -a
client :~ # cd /work1/pkg/sle15sp2_repo/
client :~ # ls -d {M,P}* | xargs -i zypper ar {} {}
client :~ # zypper ref
client :~ # zypper lr
```

---
## NIS
server
```bash
server :~ # yast
Network Services > NIS Server

# NIS Domain Name
# Slave

Netmask        Network 
255.0.0.0      127.0.0.0  
255.255.255.0  192.168.1.0 

server :~ # systemctl enable ypserv
server :~ # systemctl start ypserv
server :~ # systemctl status ypserv
server :~ # ps aux | grep ypserv
```

client
```bash
client :~ # yast
Network Services > NIS Client
# NIS Domain
# Addresses of NIS servers

client :~ # systemctl enable ypbind
client :~ # systemctl start ypbind
client :~ # systemctl status ypbind
client :~ # ps aux | grep ypbind
```

test
```bash
server :~ # useradd test1
server :~ # getent passwd
test1:x:1001:100::......
client :~ # getent passwd
(找不到test1)

● yast > Network Services > NIS server
● 原因： 當server創建新user時,ypserv的資料沒有被更新,導致ypbind拿過去的資料沒有新user
● 因此直接重啟 ypserv 的服務是沒有用的
● yast > NIS server 完成設定後的第一步為 Remove /var/yp/jjc (移除舊的檔案後續在創建新的)

server:~ # cd /var/yp; make
```

```bash
client :~ # getent passwd
client :~ # id test1
(getent passwd 有抓到使用者，但id沒有)

client :~ # reboot
```

---
## chrony
```bash
clien :~ # vi /etc/chrony.conf
server <server ip>
client :~ # systemctl enable chronyd
client :~ # systemctl start chronyd
client :~ # chronyc sources
client :~ # chronyc burst 4/4
client :~ # chronyc makestep

# check
server :~ # date
client :~ # date
```

---
## munge
server
```bash
server :~ # zypper in munge
server :~ # systemctl enable munge
server :~ # systemctl start munge
```
client
```bash
server :~ # zypper in munge
server :~ # systemctl enable munge
server :~ # systemctl start munge
```
munge.key
```bash
server :~ # scp /etc/munge/munge.key root@<client>:/etc/munge/.
server :~ # md5sum /etc/munge/munge.key
client :~ # md5sum /etc/munge/munge.key
```
check
```bash
server :~ # munge -n
server :~ # munge -n | unmunge
server :~ # munge -n | ssh <client> unmunge

client :~ # munge -n
client :~ # munge -n | unmunge
client :~ # munge -n | ssh <server> unmunge
```


---
## slurm
#### install
```bash
# server
server :~ # zypper in slurm
server :~ # systemctl enable slurmctld
server :~ # systemctl start slurmctld
server :~ # systemctl status slurmctld

# client
client :~ # zypper in slurm-node
client :~ # systemctl enable slurmd
client :~ # systemctl start slurmd
client :~ # systemctl status slurmd

● client 上 slurmd.service 等 slurm.conf 設定完成後再進行啟用
```

#### port
```bash
# server
linux :~ # firewall-cmd --add-port=6819/tcp --add-port=6818/tcp --add-port=6817/tcp --permanent
linux :~ # firewall-cmd --reload
● slurmctld: 6817/tcp
● slurmd: 6818/tcp
● slurmdbd: 6819/tcp
```

#### slurm_conf
```bash
# ClusterName
ClusterName=<cluster>                                 # ClusterName (設定 QOS 時的 ClusterName必須和這邊相同)

# BackupServer                                        # 利用 backup 功能創建2個server
SlurmctldHost=s0(192.168.1.1)                         # server 0
SlurmctldHost=s1(192.168.1.2)                         # server 1
ReturnToService=2                                     # 0: down -> idle*  1: down -> down  2: down -> idle

# port config
SlurmctldPort=6817
SlurmdPort=6818
SrunPortRange=60001-63000                             # listening ports to communicate

# User
SlurmdUser=root
SlurmUser=root

# PID
SlurmctldPidFile=/var/run/slurm/slurmctld.pid
SlurmdPidFile=/var/run/slurm/slurmd.pid

# Debug
SlurmctldDebug=debug
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=debug
SlurmdLogFile=/var/log/slurm/slurmd.log

# for slurm DB
JobCompType=jobcomp/none
JobAcctGatherType=jobacct_gather/cgroup
AccountingStorageHost=s0
AccountingStoragePort=6819
AccountingStorageUser=slurm
AccountingStoragePass=/var/run/munge/munge.socket.2

# QOS Priority Weight
AccountingStorageEnforce=limits
PriorityWeightQOS=1000                                # 此數值預設為0，若數值為0，則不啟用PriorityWeight

# Node config
# gpu
GresTypes=gpu,mps
NodeName=g01 State=idle Gres=gpu:4,mps:400 Sockets=1 CoresPerSocket=8
NodeName=g02 State=idle Gres=gpu:4 Sockets=1 CoresPerSocket=8
...

# xeon32
NodeName=s01 State=idle Sockets=2 CoresPerSocket=16
NodeName=s02 State=idle Sockets=2 CoresPerSocket=16
...

# xeon
NodeName=e01 State=idle Sockets=2 CoresPerSocket=8
NodeName=e02 State=idle Sockets=2 CoresPerSocket=8
...

# Partition config
PartitionName=xeon Nodes=e08 Default=YES MaxTime=24:00:00 State=UP
PartitionName=xeon32 Nodes=s[01-05] MaxTime=96:00:00 State=UP
PartitionName=gpu Nodes=g[01-05] MaxTime=144:00:00 State=UP
...
```

```bash
server :~ # scp /etc/slurm/slurm.conf root@<client>:/etc/slurm/.

client :~ # systemctl enable slurmd
client :~ # systemctl start slurmd

● 所有機器上的 slurm.conf 需要相同

# test
server :~ # srun -N1 hostname
```
#### QOS
```bash
# List cluster
server :~ # sacctmgr list cluster
server :~ # sacctmgr list cluster format=Cluster,QOS
# Add cluster
server :~ # sacctmgr add cluster linux                      #需要和slurm.conf內的ClusterName相同
server :~ # sacctmgr list cluster format=Cluster,QOS
```

```bash
# Add Account/User
server :~ # sacctmgr add account jjcusers cluster=linux
server :~ # sacctmgr add user name=test1 account=jjcusers cluster=linux

server :~ # sacctmgr list assoc format=Cluster,Account,User,QOS
   Cluster    Account       User        QOS
---------- ---------- ---------- ----------
     linux       root                normal
     linux       root       root     normal
     linux   jjcusers                normal
     linux   jjcusers      test1     normal
```

```bash
# Add QOS
server :~ # sacctmgr add qos jjcqos1  
server :~ # sacctmgr modify qos jjcqos1 set priority=10     # 將QOS jjcqos1 的優先級調整為10 (數字越大優先級越高)
server :~ # sacctmgr modify qos jjcqos1 set GrpJobs=2       # 將QOS jjcqos1 Job數量上限設定為2 (若submit超過2個則會等待)
server :~ # sacctmgr show qos format=name,priority,GrpJobs  # 查詢各 QOS 的 priority,GrpJobs
      Name   Priority GrpJobs
---------- ---------- -------
    normal          0
      qos1       1000
      qos2          0
   jjcqos1         10       2
   
# Modify user's QOS
server :~ # sacctmgr modify user test1 set qos=jjcqos1      # 將 test1 這個user的 QOS 設置為 jjcqos1
server :~ # sacctmgr show assoc format=cluster,user,qos     # 查詢各 user 使用的 QOS
   Cluster       User                  QOS
---------- ---------- --------------------
     linux                          normal
     linux       root               normal
     linux                          normal
     linux      test1              jjcqos1
```


#### Problem
1. slurmd 啟用時，Unable to open logfile `/var/log/slurm/slurmd.log': No such file or directory
```bash
● 原因：
1. slurmd 預設 log file 位置為 /var/log/，檢查conf內的設定為/var/log/slurm/，導致log file無法建立

● 修正：
client :~ # mkdir -p /var/log/slurm
client :~ # chown -R slurm:slurm /var/log/slurm
client :~ # systemctl restart slurmd
```

2. slurmd 啟用時，PID file /var/run/slurm/slurmd.pid not readable (yet?)
```bash
● 修正：
client :~ # vi /usr/lib/systemd/system/slurmd.service
[Service]
...
#PIDFile=/var/run/slurm/slurmd.pid                # 將本行註解掉

client :~ # systemctl daemon-reload
client :~ # systemctl restart slurmd
```

3. gpu重啟時，slurmd無法正常重啟
```bash
● 原因：
1. slurmd的pid預設生成路徑為/var/run/slurm/slurmd.pid
2. /var/run -> /run ，且df -h 發現 /run被掛載於記憶體上
3. reboot時記憶體上的資料被清除，且slurmd沒有自動重建該資料夾，導致slurmd重啟失敗

● 修正：
client :~ # vi /etc/tmpfiles.d/slurm.conf 
d  /var/run/slurm 0770 slurm slurm -

client :~ # chown -R slurm:slurm /var/log/slurm
client :~ # systemctl restart slurmd
```

4. QOS設定完成，但 GrpJobs 及 Priority 沒有確實執行
```bash
● 修正：
server :~ # vi /erc/slurm/slurn.conf
AccountingStorageEnforce=limits                   # 會強制設置 associations 和 qos
PriorityWeightQOS=1000                            # 此數值預設為0，若數值為0，則不啟用PriorityWeight

● 修改 slurm.conf 後記得更新所有機器的conf
server :~ # scp /etc/slurm/slurm.conf root@<client>:/etc/slurm/.
client :~ # systemctl restart slurmd
server :~ # systemctl restart slurmctld
```

---
## intel_compiler
```bash
● 安裝 intel_compiler 及vasp 前需要安裝一些基本套件
linux :~ # zypper in -t pattern devel_basis

● 將下載的安裝包解壓
linux :~ # tar -zxvf  parallel_studio_xe_2020_update2_cluster_edition.tgz
linux :~ # cd  parallel_studio_xe_2020_update2_cluster_edition
linux :~ # ./install.sh

● 安裝過程會經歷以下7個步驟(主要重點在5)
You will complete the following steps:
   1.  Welcome
   2.  End User License Agreement
   3.  Intel® Software Improvement Program
   4.  License Activation
   5.  Configuration
   6.  Installation
   7.  Installation Complete
```

```bash
# 4 License Activation
--------------------------------------------------------------------------------
   1. Use existing license [ default ]
   2. Activate with serial number
   3. Activate with license file, or with Intel(R) Software License Manager
   h. Help
   b. Back
   q. Quit installation
--------------------------------------------------------------------------------
# 選擇 2 使用 serial number 
Please type a selection or press "Enter" to accept default choice [ 1 ]:2
Please type your serial number (the format is XXXX-XXXXXXXX):

# 也可以選擇 3 使用 license file
預設的license file放在 /opt/intal 內，可以到先前安裝上的機器備份，需要編譯的機器必須有 license file
```

```bash
#5 Configuration
--------------------------------------------------------------------------------
   1. Accept configuration and begin installation [ default ]
   2. Customize installation
   h. Help
   b. Back
   q. Quit installation
--------------------------------------------------------------------------------
Please type a selection or press "Enter" to accept default choice [ 1 ]:2
# 選擇2進行自定義安裝

--------------------------------------------------------------------------------
   1. Accept configuration and begin installation [ default ]
   2. Change install Directory      [ /opt/intel ]
   3. Change components to install  [ All ]
   4. Change advanced options
   5. View pre-install summary
   h. Help
   b. Back
   q. Quit installation
--------------------------------------------------------------------------------
# 選擇2 更改安裝目錄
# 選擇3 更改安裝內容(不須使用的功能可以不用安裝)，C/C++、Fortran、MPI等等
--------------------------------------------------------------------------------
   1. Accept and continue [ default ]
   2. [ ] Intel Trace Analyzer and Collector 2020 Update 2
   3. [ ] Intel Cluster Checker 2019 Update 9
   4. [ ] Intel VTune Profiler 2020 Update 2
   5. [ ] Intel Inspector 2020 Update 2
   6. [ ] Intel Advisor 2020 Update 2
   7. [x] Intel C++ Compiler 19.1 Update 2
   8. [x] Intel Fortran Compiler 19.1 Update 2
   9. [x] Intel Math Kernel Library 2020 Update 2 for C/C++
   10.[x] Intel Math Kernel Library 2020 Update 2 for Fortran
   11.[ ] Intel Integrated Performance Primitives 2020 Update 2
   12.[x] Intel Threading Building Blocks 2020 Update 3
   13.[ ] Intel Data Analytics Acceleration Library 2020 Update 2
   14.[x] Intel MPI Library 2019 Update 8 (already installed)
   15.[ ] GNU* GDB 8.3
   16.[ ] Intel(R) Distribution for Python*

   Install space required:    0MB
   Space available:  1.7TB

   a. Select/unselect all
   h. Help
   b. Back
   q. Quit installation
--------------------------------------------------------------------------------
# 設定完成後 back到以下畫面 選擇1 開始安裝
--------------------------------------------------------------------------------
   1. Accept configuration and begin installation [ default ]
   2. Customize installation

   h. Help
   b. Back
   q. Quit installation
--------------------------------------------------------------------------------
Please type a selection or press "Enter" to accept default choice [ 1 ]: 
```

source
```bash
linux :~ # vi intelvar.sh
INTEL_PARALLEL_STUDIO_PATH=/<path to your install directory>/compilers_and_libraries_2020.2.254/linux

source $INTEL_PARALLEL_STUDIO_PATH/bin/compilervars.sh intel64
source $INTEL_PARALLEL_STUDIO_PATH/mkl/bin/mklvars.sh intel64
source $INTEL_PARALLEL_STUDIO_PATH/mpi/intel64/bin/mpivars.sh
```

---
## nvidia
到nvidia官網下載以下安裝包
1. NVIDIA HPC SDK >>>[NVIDIA HPC SDK](https://developer.nvidia.com/nvidia-hpc-sdk-releases) 
2. CUDA Toolkit>>>[CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit-archive) 
```bash
linux :~ # ls -l
nvhpc-2021-21.3-1.suse.x86_64.rpm
nvhpc-21-3-21.3-1.suse.x86_64.rpm
cuda-repo-sles15-11-3-local-11.3.0_465.19.01-1.x86_64.rpm

# install
linux :~ # rpm -ivh nvhpc-2021-21.3-1.suse.x86_64.rpm  nvhpc-21-3-21.3-1.suse.x86_64.rpm
linux :~ # rpm -ivh cuda-repo-sles15-11-3-local-11.3.0_465.19.01-1.x86_64.rpm
linux :~ # zypper in cuda                # 跑vasp_gpu時需要 nvidia-computeG05
linux :~ # zypper in gcc-fortran         # 後續編譯vasp需要
linux :~ # zypper in libncurses5         # 後續編譯vasp需要
  
```

source
```bash
linux :~ # vi hpc.sh
NVARCH=`uname -s`_`uname -m`; export NVARCH
NVCOMPILERS=/opt/nvidia/hpc_sdk; export NVCOMPILERS
MANPATH=$MANPATH:$NVCOMPILERS/$NVARCH/21.3/compilers/man; export MANPATH
PATH=$NVCOMPILERS/$NVARCH/21.3/compilers/bin:$PATH; export PATH

export CUDA_ROOT=/opt/nvidia/hpc_sdk/Linux_x86_64/21.3/compilers
export CUDA_PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/21.3/cuda/11.2/targets/x86_64-linux
export CUDA_MATH=/opt/nvidia/hpc_sdk/Linux_x86_64/21.3/math_libs/11.2/targets/x86_64-linux

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/hpc_sdk/Linux_x86_64/21.3/cuda/11.2/targets/x86_64-linux/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/hpc_sdk/Linux_x86_64/21.3/math_libs/11.2/targets/x86_64-linux/lib
```


---
## vasp
1. 本部分的編譯皆有加入 vtstcode及 solvent進行編譯
2. 到vtst官網下載vtstcode(用於編譯NEB等功能)>>>[vtstcode下載連結](http://theory.cm.utexas.edu/vtsttools/download.html) 
3. 編譯時盡量在之後要執行的機器上編譯，環境會比較接近
4. 編譯前需要 source intel 和 nvidia(only for gpu) 的相關變數

#### vasp_std
`vasp5.4.4`
```bash
linux :~ # tar -zxvf vasp.5.4.4.tgz
linux :~ # tar -zxvf vtstcode-180.tgz         #解壓縮vtstcode
linux :~ # cp vtstcode-180/* vasp.5.4.4/src/. #把vtstcode複製到vasp/src內
```

```bash
# 修改src/main.F
linux :~ # vi main.F
將
CALL CHAIN_FORCE(T_INFO%NIONS,DYN%POSION,TOTEN,TIFOR, &
     LATT_CUR%A,LATT_CUR%B,IO%IU6)
改為
CALL CHAIN_FORCE(T_INFO%NIONS,DYN%POSION,TOTEN,TIFOR, &
     TSIF,LATT_CUR%A,LATT_CUR%B,IO%IU6)
```

```bash
# 修改src/.objects
linux :~ # vi .objects
找到chain.o (第一個)並在前面加入
bfgs.o dynmat.o  instanton.o  lbfgs.o sd.o   cg.o dimer.o bbm.o \
fire.o lanczos.o neb.o  qm.o opt.o

例如
bfgs.o dynmat.o  instanton.o  lbfgs.o sd.o   cg.o dimer.o bbm.o \
fire.o lanczos.o neb.o  qm.o opt.o \
chain.o
```

```bash
# 增加solvent
linux :~ # vi makefile.include
CPP_OPTIONS 加入 -Dsol_compat
```

```bash
# source and make
linux :~ # source intelvar.sh
linux :~ # which icc ifort mpicc mpifc
linux :~ # make std
```

`vasp6.1.1`
```bash
● vasp_std的編譯與第5版的方式大致相同，參考vasp5的編譯即可
linux :~ # tar -zxvf vasp.6.1.1.tgz
linux :~ # tar -zxvf vtstcode-180.tgz         #解壓縮vtstcode
linux :~ # cp vtstcode-180/* vasp.6.1.1/src/. #把vtstcode複製到vasp/src內
# 修改src/main.F
# 修改src/.objects
# 修改makefile.include，加入 Dsol_compat
```

#### vasp_gpu
`vasp5.4.4  vasp6.1.1`
```bash
● vasp_gpu的編譯與std大致相同，但makefile.include以 及.objects 的部分需要進行修改
```

```bash
# 修改src/.objects
linux :~ # vi .objects
找到chain.o (第二個)並在前面加入
bfgs.o dynmat.o  instanton.o  lbfgs.o sd.o   cg.o dimer.o bbm.o \
fire.o lanczos.o neb.o  qm.o opt.o

例如
bfgs.o dynmat.o  instanton.o  lbfgs.o sd.o   cg.o dimer.o bbm.o \
fire.o lanczos.o neb.o  qm.o opt.o \
chain.o

● 本部分與vasp_std加入的部分不相同，原因如下：
● 第一個 chain. o 出現在 SOURCE=\ 下，第二個 chain.o出現在 SOURCE_GPU=\ 下
● 編譯 vasp_gpu 時會使用到SOURCE_GPU下方的 objects，因此需要加在第二個chain.o的位置
```

```bash
# 修改 makefile.include
linux :~ # vi makefile.include

將
INCS       =-I$(MKLROOT)/include/fftw
修改為
INCS       =-I$(MKLROOT)/include/fftw -I$(CUDA_ROOT)/include_acc

將
CC         = icc
修改為
CC         = icc -I$(CUDA_PATH)/include -I$(CUDA_MATH)/include

將
CUDA_LIB   := -L$(CUDA_ROOT)/lib64 -lnvToolsExt -lcudart -lcuda -lcufft -lcublas
修改為
CUDA_LIB   = -L$(CUDA_PATH)/lib -lnvToolsExt -lcudart -lcuda -L$(CUDA_MATH)/lib -lcufft -lcublas
● 修改cuda lib

將
CFLAGS     = -fPIC -DADD_ -Wall -openmp -DMAGMA_WITH_MKL -DMAGMA_SETAFFINITY -DGPUSHMEM=300 -DHAVE_CUBLAS
修改為
CFLAGS     = -fPIC -DADD_ -Wall -qopenmp -DMAGMA_WITH_MKL -DMAGMA_SETAFFINITY -DGPUSHMEM=300 -DHAVE_CUBLAS
● 較新版的intel compiler可能不支援openmp，依照編譯時的error修改成qopenmp

將
MPI_INC    = $(I_MPI_ROOT)/include64/
修改為
MPI_INC    = $(I_MPI_ROOT)/intel64/include
● 不同版本的 intel compiler可能有不同的資料夾型態，修改成符合的型態即可


將以下的gencode=arch=compute_30這行給刪除 # vasp6可能需要用告高的進行編譯，依照error message除錯即可
GENCODE_ARCH    := -gencode=arch=compute_30,code=\"sm_30,compute_30\" \
                   -gencode=arch=compute_35,code=\"sm_35,compute_35\" \
                   -gencode=arch=compute_60,code=\"sm_60,compute_60\"
● 在較高版本的cuda當中已經將30給移除，在vasp6編譯時也會遇到類似的狀況

● CUDA_ROOT、CUDA_PATH、CUDA_MATH 寫於/opt/nvidia/hpc.sh中

```

```bash
# source and make
linux :~ # source intelvar.sh
linux :~ # source hpc.sh
linux :~ # which icc ifort mpicc mpifc nvcc
linux :~ # make gpu
```

#### problem

1. gpu版本加入vtstcode時編譯失敗，(neb.o等等編譯被跳過)
```bash
● 原因：
1. 加入vtst編譯時需要修改.object，並在chain.o前加入vtstcode的objects
2. 檔案內有2個chain.o，第一個位於SOURCE=\ 下，第二個 chain.o出現在 SOURCE_GPU=\ 下
3. 編譯CPU版本時會吃到SOURCE=\ 下的參數，編譯GPU版本會吃到SOURCE_GPU=\ 下的參數

● 修正：
1. 檢查.object修改的位置是否正確
```

2. gpu編譯完成後運行時發生以下錯誤
```bash
linux :~ # CUDA Error in cuda_fft.cu, line 323: invalid device function Failed to execute cuda_fftwav!
```

```bash
● 原因：
1. 各顯卡的 Compute Capability 不同，以K80為例，	Compute Capability 為3.7
2. K80若編譯時使用 GENCODE_ARCH    := -gencode=arch=compute_60,code=\"sm_60,compute_60\" ，則會出現本錯誤

● 修正：
1. 修改makefile中的GENCODE_ARCH參數，使其符合各GPU的Compute Capability
註：K80使用 GENCODE_ARCH    := -gencode=arch=compute_35,code=\"sm_35,compute_35\ 運行正常
```
關於顯卡Compute Capability，可參考此連結 >>>[GPU Compute Capability](https://blog.csdn.net/Allyli0022/article/details/54628987?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control) 
