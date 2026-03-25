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
- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
- [ ] 映像列表：`sudo docker images` 輸出

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | （時間點） | （此節點代表的狀態） | （列出建點前做了哪些驗證） |
| docker-ready | （時間點） | （此節點代表的狀態） | （列出建點前做了哪些驗證） |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | （填入） |
| apt-cache policy 有候選版本 | 是 | 否 | （填入） |
| docker 重裝可行 | 是 | 否 | （填入） |
| hello-world 成功 | 是 | N/A | （填入） |
| nginx curl 成功 | 是 | N/A | （填入） |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | （你的實測） | （你的實測） |
| 適用情境 | （你的判斷） | （你的判斷） |
| 風險 | （你的判斷） | （你的判斷） |

## Snapshot 保留策略
- 新增條件：
- 保留上限：
- 刪除條件：

## 最小可重現命令鏈
（列出讓他人能重現故障注入與回復驗證的命令序列）

## 排錯紀錄
- 症狀：
- 診斷：（你首先查了什麼？）
- 修正：（做了什麼改動？）
- 驗證：（如何確認修正有效？）

## 設計決策
（說明本週至少 1 個技術選擇與取捨）
