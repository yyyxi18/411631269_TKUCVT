# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.40.130 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.8.128 | — |
| app | 應用層 | NIC 1 | Host-only | 192.168.8.129 | SSH from 192.168.56.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.8.130 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：ED25519
- 公鑰部署到：
  - app：~/.ssh/authorized_keys
  - db：~/.ssh/authorized_keys
- 免密碼登入驗證：
  - bastion → app：app
  - bastion → db：db

## 防火牆規則

### app 的 ufw status
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    192.168.8.0/24
```

### db 的 ufw status
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    192.168.8.129             
22/tcp                     ALLOW IN    192.168.8.128
```

### 防火牆確實在擋的證據
```
curl: (28) Connection timed out
```

## ProxyJump 跳板連線
- 指令：
```
Host bastion
    HostName 192.168.8.128
    User yyyxi

Host app
    HostName 192.168.8.129
    User yyyxi
    ProxyJump bastion

Host db
    HostName 192.168.8.130
    User yyyxi
    ProxyJump bastion
```
- 驗證輸出：
ssh app "hostname" 輸出 app
ssh db "hostname" 輸出 db
- SCP 傳檔驗證：
  <img width="786" height="115" alt="image" src="https://github.com/user-attachments/assets/111c65ec-60b4-4b1c-beb5-9f165ebc48ac" />
  <img width="803" height="114" alt="image" src="https://github.com/user-attachments/assets/2d7f3b1a-4926-4318-bb69-aa95e7a4c991" />

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | active + allow SSH |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **timed out** | 成功 |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **refused** | 成功 |

## timeout vs refused 差異
- Connection timed out
  表示封包被防火牆丟棄（drop），系統沒有任何回應。
  排錯方向應從防火牆（ufw）或網路 ACL 檢查。
- Connection refused
  表示目標主機有回應，但該 port 沒有服務在監聽（回傳 TCP RST）。
  排錯方向應檢查服務狀態（如 sshd 是否啟動）。

## 網路拓樸圖
<img width="552" height="990" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/828b5f26-54c9-4fb0-8d76-d282bc0c5ba0" />

## 排錯紀錄
- 症狀：
  - SSH 無法連線（timeout / refused）
  - curl 無法存取服務
- 診斷：
  - 先測 ping（L3）
  - 再測 SSH（L4）
  - 使用 ss -tlnp 檢查服務
  - 使用 ufw status 檢查防火牆
- 修正：
  - 修正 UFW 規則（允許 192.168.8.0/24）
  - 啟動 SSH：systemctl start ssh
  - 修正 SSH config（ProxyJump）
- 驗證：
  - ssh app 成功
  - ssh db 成功
  - scp 可正常傳檔
  - HTTP 服務可回應

## 設計決策
本架構採用 bastion 作為唯一對外入口，將 app 與 db 放置於 Host-only 私有網段中，以達到最小暴露原則。
其中 db 雖允許 bastion 直接連線，而非僅允許從 app 轉跳，主要考量為：
- 管理與除錯便利性（可直接維護資料庫）
- 減少多層跳轉造成的複雜度

但此設計在實務上可進一步強化為：
- db 僅允許來自 app 的連線
- bastion 不直接接觸資料層
此為安全性與維運便利性之間的取捨。
