# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.40.130 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.8.128 | 內網互連 |
| server-b | NIC 1 |Host-only | 192.168.8.129 | 內網互連 |

## 連線驗證紀錄

- [X] dev-a NAT 可上網：`ping google.com` 輸出
      <img width="1028" height="306" alt="image" src="https://github.com/user-attachments/assets/b123c031-31fa-4a2b-95c6-a4e72ab126a8" />
- [X] 雙向互 ping 成功：貼上雙方 `ping` 輸出
      <img width="925" height="303" alt="server-b" src="https://github.com/user-attachments/assets/8a9d67bb-f19e-400d-8e59-e11b6193cb86" />
      <img width="961" height="303" alt="dev-a" src="https://github.com/user-attachments/assets/74bd464f-8be5-444a-97ca-71f095b47098" />
- [X] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
      <img width="1112" height="1106" alt="螢幕擷取畫面 2026-04-08 162053" src="https://github.com/user-attachments/assets/e65efe55-5d8d-4fa9-8f72-6596f4bb6d36" />
- [X] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
      <img width="987" height="201" alt="image" src="https://github.com/user-attachments/assets/d838e4fc-5d87-4364-b419-bf76da68a505" />
- [X] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
      <img width="1115" height="156" alt="螢幕擷取畫面 2026-04-08 163018" src="https://github.com/user-attachments/assets/add0824f-c6d4-4afe-95b3-d575876477c1" />

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | UP |
| dev-a ping server-b | 成功 | 失敗 | 成功 |
| dev-a SSH server-b | 成功 | 失敗 | 成功 |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| dev-a ping server-b | 成功 | 成功 | 成功 |
| dev-a SSH server-b | 成功 | Connection refused | 成功 |

## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）
下面這版可直接交。
### 1. L2：先確認網卡與連線狀態
L2 主要檢查網路介面是否存在、是否為正確的介面、以及介面是否處於 **UP** 狀態。若 Host-only 介面已被停用，則後續 L3 與 L4 都會一併失敗。

#### 檢查命令
```bash
ip address show
ip link show
```

#### 觀察重點
* 是否看得到正確網卡名稱（例如 `ens33`、`ens37`）
* 是否為 `state UP`
* 是否因故障注入變成 `state DOWN`

### 本次使用情境
* 在 `server-b` 上使用：

```bash
sudo ip link set ens33 down
ip address show ens33
```

#### 判斷
* 若介面 **DOWN**：屬於 L2 問題
* 修復方式：
```bash
sudo ip link set ens33 up
```

### 2. L3：確認 IP 與路由是否正常

當 L2 正常後，接著確認是否取得正確 IP、是否與對端位於同一網段，以及路由是否存在。若沒有正確 IP 或路由錯誤，則即使網卡 UP 也無法互通。

#### 檢查命令

```bash
ip address show
ip route show
ping -c 4 <對方IP>
```

#### 觀察重點
* 是否有正確 IP

  * `dev-a Host-only`：`192.168.8.128`
  * `server-b Host-only`：`192.168.8.129`
* 是否同屬 `192.168.8.0/24`
* `ping` 是否成功
* 是否有封包遺失或 `No route to host`

#### 本次使用情境
```bash
ip route show
ping -c 4 192.168.8.129
ping -c 4 192.168.8.128
```

#### 判斷
* 若 `ping` 失敗、出現 `100% packet loss` 或 `No route to host`：優先回頭檢查 L2 與 L3
* 若 `ping` 成功：代表 L2、L3 大致正常，可以進入 L4

### 3. L4：確認服務是否在監聽與可連線
在 L2、L3 都正常後，最後才檢查 SSH 服務。此時若 `ping` 可通但 SSH 不通，通常就是 L4 或應用服務層的問題，例如 SSH 沒啟動、沒有監聽 port 22，或被 socket activation 影響。

#### 檢查命令
```bash
ss -tlnp | grep :22
systemctl status ssh --no-pager
ssh <user>@<ip> "hostname"
```

#### 觀察重點
* 是否有 `0.0.0.0:22` 或 `[::]:22`
* `ssh.service` 是否正常執行
* 從 `dev-a` 連到 `server-b` 是否成功執行遠端指令

#### 本次使用情境
```bash
ss -tlnp | grep :22
ssh yyyxi@192.168.8.129 "hostname"
```

#### 判斷
* 若 `ping` 成功但 SSH 失敗：
  * `Connection refused` → SSH 服務未監聽
  * `No route to host` → 問題其實還在 L2/L3
* 若 `ssh ... "hostname"` 輸出 `server-b`：表示 SSH 正常

### 總結排錯流程
#### 第一步：L2
先檢查網卡是否存在且為 `UP`
```bash
ip address show
ip link show
```

#### 第二步：L3
再檢查 IP、路由、以及 ping 是否成功
```bash
ip address show
ip route show
ping -c 4 <對方IP>
```

#### 第三步：L4
最後檢查 SSH port 22 是否監聽，以及遠端連線是否可用
```bash
ss -tlnp | grep :22
systemctl status ssh --no-pager
ssh <user>@<ip> "hostname"
```

## 網路拓樸圖
<img width="2297" height="848" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/3944c269-d96f-413f-836c-465d987e66b9" />

## 排錯紀錄
- 症狀：
  1. ping 失敗（100% packet loss）
  2. ssh 失敗，顯示 No route to host
- 診斷：
  從 L2 開始檢查：
  ```
  ip address show
  ```
  發現：ens33 為 state DOWN，判斷為 網卡介面被關閉（L2 問題）
- 修正：
  在 server-b 啟用網卡：
  ```
  sudo ip link set ens33 up
  sleep 5
  ```
- 驗證：
  1. 確認介面恢復
     ```
     ip address show ens33
     ```
  2. dev-a 測試
     ```
     ping -c 4 192.168.8.129
     ```
     ```
     ssh yyyxi@192.168.8.129 "hostname"
     ```

# 設計決策
## 決策內容
* **dev-a**：配置 NAT + Host-only
  → 同時具備「對外上網」與「內網通訊」能力
* **server-b**：僅配置 Host-only
  → 僅允許內網存取，不直接連外

## 設計理由
### 1. 模擬真實架構（分層設計）
此配置對應實務中的常見架構：
* dev-a：類似 **跳板機 / Gateway**
* server-b：類似 **內網服務主機（Backend Server）**
server-b 不直接暴露在外網，提高安全性

### 2. 強化網路分層觀念
透過限制 server-b 僅在 Host-only 網段：
* L2 / L3：只能在 192.168.8.0/24 溝通
* L4：服務（SSH）僅內網可存取
有助於清楚觀察不同層級的故障影響

### 3. 支援故障注入實驗
此設計能清楚區分不同類型問題：
* 關閉網卡（L2） → ping / ssh 全失敗
* 關閉 SSH（L4） → ping 成功但 ssh 失敗
若 server-b 同時有 NAT，測試結果會變得模糊

## 取捨（Trade-off）
### 優點
* 安全性較高（server-b 不對外）
* 架構清晰（內外網分離）
* 易於觀察與排錯（分層明確）

### 缺點
* server-b 無法直接上網（例如 apt install）
* 需透過 dev-a 或額外配置才能取得外部資源

## 結論
本設計透過讓 server-b 僅存在於 Host-only 網段，達到「**安全隔離 + 分層排錯清晰化**」的目的，雖然犧牲了對外連線能力，但能更有效地支援本實驗的學習目標與故障分析。

