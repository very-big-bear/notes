## Vasp編譯

 ```bash
 基本安裝
 linux :~# zypper in gcc
 linux :~# zypper in -t pattern devel_basis
 ```
 
 ```bash
 安裝intel compiler
 執行sh檔案
 ```
 
 ```bash
 到mpicc，ifort，mpi90
 linux :~# source ~
 檢查
 linux :~# which mpicc
 linux :~# which ifort
 linux :~# which mpif90
 解壓縮vasp
 linux :~# tar zxvf vasp.....
 linux :~# cd /build/std
 linux :~# cp arch/makefile.include.linux_intel makefile.include
 linux :~# make std
 ```
