# 基本docker架設使用
---

## 安裝開啟服務
```bash
linux :~# zypper in docker
linux :~# systemctl start docker.service
```
## 查詢repository
```bash
linux :~# docker search <關鍵字>
上google查docker alpine (建議這個，比較精準)
```
## 基本localhost的repository查詢
```bash
linux :~# docker ps -a
linux :~# docker ps
linux :~# docker images

REPOSITORY     TAG      IMAGEID     CREATED     SIZE
alpine         latest   ---         4weeks ago  5.59MB
ubuntu         latest   ---         2months ago 72.8MB
```

## 安裝repository、使用
```bash
linux :~# docker pull <查到的repository>
linux :~# docker run -itd <查到的repository>
linux :~# docker run -itd -v <localhost資料夾位置>:<docker資料夾位置> <查到的repository> (連結2邊 資料夾)
linux :~# docker exec -it <名稱> sh --
```
