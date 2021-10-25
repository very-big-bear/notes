# 基本golang安裝

## centos
```bash
centos:~ # yum install golang
```

---
## ubuntu
```bash
ubuntu:~ # apt install golang
```

---
## linux
```bash
linux:~ # wget https://golang.org/dl/go1.17.2.linux-amd64.tar.gz
linux:~ # tar zxf go1.17.2.linux-amd64.tar.gz
linux:~ # export PATH=/user/local/go/bin:&PATH
```

---
## windows
```bash
在https://golang.org/dl/ 網站下載 go1.17.2.windows-amd64.zip (選最新版)
解壓縮檔案
在控制台找編輯系統環境變數，點選環境變數，在系統環境變數下的Path新增一個路徑，此路徑為解壓後檔案內bin的位置
ex: D:\GO\go1.17.2.windows-amd64\go\bin
