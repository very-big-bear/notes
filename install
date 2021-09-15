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
