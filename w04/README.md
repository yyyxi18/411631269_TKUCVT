# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑           | FHS 定義               | Docker 用途                          |
| ---------------- | -------------------- | ---------------------------------- |
| /etc/docker/     | 系統層級服務設定檔目錄          | 存放 Docker daemon 設定（如 daemon.json） |
| /var/lib/docker/ | 應用程式持久化資料目錄          | 存放 image、container、volume 等資料      |
| /usr/bin/docker  | 使用者可執行程式路徑           | Docker CLI 執行檔                     |
| /run/docker.sock | 系統 runtime socket 檔案 | CLI 與 Docker daemon 溝通的 API 入口     |


## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：236K
- 拉取映像後 /var/lib/docker/ 大小：240K

## 權限結構

### Docker Socket 權限解讀
```
srw-rw---- 1 root docker 0 Apr 12 14:59 /var/run/docker.sock
```
- s：socket 類型
- owner：root（可讀寫）
- group：docker（可讀寫）
- others：無權限

### 使用者群組
```
uid=1000(yyyxi) gid=1000(yyyxi) groups=1000(yyyxi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
```
初始狀態未包含 docker 群組，因此無法直接使用 docker CLI。

### 安全意涵
將使用者加入 docker 群組後，即可直接控制 Docker daemon。由於 daemon 以 root 權限執行，使用者可透過 bind mount 等方式存取 host 上的敏感檔案（如 /etc/shadow）。本實驗已成功將 /etc/shadow 掛入容器並讀取其內容，證明 docker 群組實質上具備接近 root 的權限。因此 docker 群組應視為高權限群組，需謹慎控管。

## 程序與服務管理

### systemctl status docker
```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled)
     Active: active (running)
     Main PID: 1602 (dockerd)
```

### journalctl 日誌分析
- Starting docker.service
- Docker daemon has completed initialization
- API listen on /run/docker.sock
- image pulled nginx:latest
- container create / delete event
可觀察 daemon 啟動流程、image 拉取與 container lifecycle 事件。

### CLI vs Daemon 差異
Docker CLI（/usr/bin/docker）是使用者端工具，而 Docker daemon（dockerd）為背景服務。CLI 透過 /run/docker.sock 與 daemon 溝通。即使 daemon 停止，docker --version 仍可正常執行，因為該指令不需與 daemon 溝通；但 docker ps、docker run 等操作會失敗。因此 CLI 正常不代表 Docker 系統可用。

## 環境變數

- $PATH：/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
- which docker：/usr/bin/docker
- 容器內外環境變數差異觀察：
  Host 端包含 USER=yyyxi、HOME=/home/yyyxi、SHELL=/bin/bash；容器內通常為 root 環境（HOME=/root），且變數較精簡。說明 container 與 host 環境彼此隔離。

## 故障場景一：停止 Docker Daemon

| 項目                      | 故障前       | 故障中             | 回復後       |
| ----------------------- | --------- | --------------- | --------- |
| systemctl status docker | active    | inactive (dead) | active    |
| docker --version        | 正常        | 正常              | 正常        |
| docker ps               | 正常        | Cannot connect  | 正常        |
| ps aux grep dockerd     | 有 process | 無 process       | 有 process |

## 故障場景二：破壞 Socket 權限

| 項目                      | 故障前        | 故障中               | 回復後        |
| ----------------------- | ---------- | ----------------- | ---------- |
| ls -la docker.sock 權限   | srw-rw---- | srw-------        | srw-rw---- |
| docker ps（不加 sudo）      | 正常         | permission denied | 正常         |
| sudo docker ps          | 正常         | 正常                | 正常         |
| systemctl status docker | active     | active            | active     |


## 錯誤訊息比較

| 錯誤訊息                                | 根因           | 診斷方向                          |
| ----------------------------------- | ------------ | ----------------------------- |
| Cannot connect to the Docker daemon | daemon 未執行   | 檢查 `systemctl status docker`  |
| permission denied…docker.sock       | 權限不足（socket） | 檢查 `/var/run/docker.sock` 與群組 |
前者表示服務不存在或不可達，後者表示服務存在但拒絕存取。

## 排錯紀錄
- 症狀：docker 指令無法執行（permission denied / cannot connect）
- 診斷：先檢查 systemctl status docker、groups、ls -l docker.sock
- 修正：加入 docker 群組、修正 socket 權限、啟動 daemon
- 驗證：docker ps、docker run hello-world 正常

## 設計決策
本實驗選擇使用 usermod -aG docker 將使用者加入 docker 群組，而非每次使用 sudo docker。此方式可提升操作便利性，避免每次執行都需輸入密碼。然而其風險在於 docker 群組具備接近 root 的權限，若使用者帳號遭入侵，攻擊者可透過 Docker 控制 host 系統。因此在實務環境中，應限制 docker 群組成員數量，或改用 rootless Docker 等機制降低風險。
