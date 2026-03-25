# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：TKUVCT_411631269
- Ubuntu 版本：
```
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.4 LTS
Release:	24.04
Codename:	noble
```
- Docker 版本：sudo docker --version
- Docker Compose 版本：Docker Compose version v5.1.1

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s):2|
| 記憶體 | 4 GB | `free -h \| grep Mem` |Mem:7.7Gi 1.9Gi 2.8Gi 43Mi 3.2Gi  5.8Gi  |
| 磁碟 | 40 GB | `df -h /` |/dev/sda2    59G   11G   46G  19% /|
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor vendor = VMware|

## 四層驗收證據
- [ ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
<img width="1207" height="241" alt="image" src="https://github.com/user-attachments/assets/240ccf8a-19b4-4ff2-8f37-4eba063856c0" />

- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出
<img width="1221" height="299" alt="image" src="https://github.com/user-attachments/assets/81279508-a9df-4a57-b8af-de3d7f255d34" />

- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
<img width="1222" height="765" alt="image" src="https://github.com/user-attachments/assets/1891fef8-e881-4206-940e-c399d5ecc4bb" />

- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
<img width="966" height="603" alt="image" src="https://github.com/user-attachments/assets/4e9ca13f-f6d6-4656-85c0-965a252ae5b7" />

- [ ] Compose：`docker compose version` 可執行
<img width="608" height="58" alt="image" src="https://github.com/user-attachments/assets/6b68a2b4-0c39-42ce-9eaa-903963a4fba8" />

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
`4b591a506c97fe1ad130dcb57a542a2e06206b1436fce086c0506bf63e52633c`
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
`/ # `
- [ ] 映像列表：`sudo docker images` 輸出
<img width="1214" height="187" alt="image" src="https://github.com/user-attachments/assets/a5c85a66-6c05-4207-838f-1a999d037603" />


## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | Docker 安裝完成後 | 建立第一個環境健康的可回復基線| hostnamectl、ip route、docker --version、docker compose version、systemctl status docker、docker run --rm hello-world |
| docker-ready | 完成 container 測試後 | Docker 環境與 container 功能驗證完成，之後可以繼續做 | sudo systemctl status、docker --no-pager、sudo docker run --rm hello-world、sudo docker images |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 10-20分鐘 | 1-2分鐘，關機然後選snapshot再開機而已|
| 適用情境 | 容易修復的錯誤|系統死機動不了的時候|
| 風險 | 若操作錯誤可能造成更多問題 | 會遺失snapshot之後所有改變 |

## Snapshot 保留策略
- 新增條件：在系統環境健康或完成重要設定後建立，例如完成docker安裝並確認服務正常
- 保留上限：3–5個snapshot，避免磁碟空間被大量占用
- 刪除條件：當新的健康snapshot建立後，可刪除較舊且不再需要回復的snapshot

## 最小可重現命令鏈
```
# 故障前基線
echo "=== 故障前 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10

# 注入故障
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

echo "=== 故障中 ==="
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10
sudo apt -y install docker-ce 2>&1 | tail -5

sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
sudo apt update
apt-cache policy docker-ce | head -5

sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

sudo poweroff

echo "=== 回復後 ==="
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/docker.list
sudo apt update

sudo systemctl status docker --no-pager
sudo docker --version
docker compose version
sudo docker run --rm hello-world
sudo docker images

free -h
df -h /
```

## 排錯紀錄
- 症狀：執行 apt-cache policy docker-ce 時，只顯示 /var/lib/dpkg/status，Docker repository 消失
- 診斷：檢查 /etc/apt/sources.list.d/ 發現 docker.list 被改名為 docker.list.broken，APT因副檔名不正確而忽略repository
- 修正：將 docker.list.broken 改回 docker.list 並重新執行 apt update
- 驗證：重新執行 apt-cache policy docker-ce，確認再次出現 https://download.docker.com/linux/ubuntu noble/stable

## 設計決策
這次實驗使用修改repository檔名作為故障注入方式，因為這個方法不會破壞docker已安裝的套件，只會影響APT套件來源，因此能安全的模擬repository設定錯誤的情境。 而且老師說這個錯誤很容易回復，只需將檔名改回原本名稱即可恢復系統狀態。
