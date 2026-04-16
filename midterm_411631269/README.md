# 期中實作 — 411631269 姚育祺

## 1. 架構與 IP 表
| VM      | 角色            | 介面      | 模式           | IP/Mask           | 網段              | 用途                  |
| ------- | ------------- | ------- | ------------ | ----------------- | --------------- | ------------------- |
| bastion | 跳板機           | ens33   | NAT          | 192.168.40.130/24 | 192.168.40.0/24 | 對外連線、更新套件、對外 SSH 測試 |
| bastion | 跳板機           | ens37   | Host-only    | 192.168.8.128/24  | 192.168.8.0/24  | 內部管理網路，連到 app       |
| app     | 應用主機          | ens33   | Host-only    | 192.168.8.129/24  | 192.168.8.0/24  | 與 bastion 內網互通      |
| bastion | Docker bridge | docker0 | Linux bridge | 172.17.0.1/16     | 172.17.0.0/16   | Docker 預設橋接網路       |
| app     | Docker bridge | docker0 | Linux bridge | 172.17.0.1/16     | 172.17.0.0/16   | Docker 預設橋接網路       |
<img width="2085" height="1415" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/dca3e5d4-403e-490e-bd73-e9d895d5bc73" />

## 2. Part A：VM 與網路
### IP 表
| VM | 角色 | 網卡 | 模式 | IP |
|----|------|------|------|----|
| bastion | 跳板機 | ens33 | NAT | 192.168.40.130 |
| bastion | 跳板機 | ens37 | Host-only | 192.168.8.128 |
| app | 應用層 | ens33 | Host-only | 192.168.8.129 |

### 連通測試
#### app → bastion
```
ping 192.168.8.128
```
#### bastion → app
```
ping 192.168.8.129
```

## 3. Part B：金鑰、ufw、ProxyJump
### 防火牆規則表
| VM      | 預設政策          | 規則                          |
| ------- | ------------- | --------------------------- |
| bastion | deny incoming | allow 22/tcp                |
| app     | deny incoming | allow 22 from 192.168.8.128 |

### ProxyJump設定
```
Host bastion
    HostName 192.168.40.130
    User yyyxi

Host app
    HostName 192.168.8.129
    User yyyxi
    ProxyJump bastion
```
### ssh app 成功證據
```
ssh app
```
<img width="1693" height="594" alt="image" src="https://github.com/user-attachments/assets/a8b03f5a-ac3d-47ec-b26d-204e5d6eee60" />

## 4. Part C：Docker 服務
```
systemctl status docker
```
<img width="1735" height="531" alt="image" src="https://github.com/user-attachments/assets/81f846cc-d979-4b17-9538-f9944275191d" />

```
curl -I http://192.168.8.129:8080
```
<img width="560" height="229" alt="image" src="https://github.com/user-attachments/assets/c07e3e30-6821-41a4-b090-e3e14ddc7730" />

## 5. Part D：故障演練
### 故障 1：F3（Docker 停止）
- 注入方式：
  ```
  sudo systemctl stop docker
  sudo systemctl stop docker.socket
  ```
- 故障前：
  ```
  sudo docker ps
  curl -I http://192.168.8.129:8080
  ```
- 故障中：<img width="999" height="72" alt="image" src="https://github.com/user-attachments/assets/68c32d0b-120a-46aa-939a-fd9c8969dc92" />
         <img width="1029" height="45" alt="image" src="https://github.com/user-attachments/assets/2b9ddf18-9d43-498a-9271-93110b24a012" />
- 回復後：
  <img width="1376" height="178" alt="image" src="https://github.com/user-attachments/assets/f4c174e1-7b06-407e-bcd1-792a70ee1993" />
- 診斷推論：SSH 正常但 HTTP 服務失效，且 Docker daemon 無法連線，顯示問題發生於應用服務層，而非網路層。

### 故障 2：F1（網卡關閉）
- 注入方式：
  ```
  sudo ip link set ens33 down
  ```
  
- 故障前：
  ```
  ping 192.168.8.129
  ssh app
  ```
  
 - 故障中：
    <img width="1179" height="268" alt="image" src="https://github.com/user-attachments/assets/11597e4f-6bf0-4b61-9a81-cf8ba3f942ba" />

 - 回復後：
    <img width="1167" height="144" alt="image" src="https://github.com/user-attachments/assets/d87551f6-aff8-43f9-a824-ed11a9a82164" />

 - 診斷推論：網卡關閉導致封包無法傳輸，ping 與 ssh 均失敗，屬於 L2/L3 網路層問題，而非防火牆或服務層。

### 症狀辨識（若選 F1+F2 必答）
兩個都 timeout，我怎麼分？
| 測試     | F1 | F3          |
| ------ | -- | ----------- |
| ping   | 不通 | 可通          |
| ssh    | 不通 | 可通          |
| curl   | 不通 | 不通          |
| docker | 無關 | daemon 無法連線 |
判斷方式：若 ping 不通，代表問題發生於 L2/L3 網路層；若 ping 正常但服務無法回應，則為應用服務層問題。

## 6. 反思（200 字）
本次實驗讓我更清楚理解分層隔離的重要性。在過去遇到連線失敗時，容易直接判斷為整體系統異常，但透過本次實作，可以明確區分不同層級的問題來源。在 F1 中，網卡關閉導致 ping 與 ssh 同時失效，表示問題發生在網路層；而在 F3 中，ssh 仍可正常登入，但應用服務無法回應，顯示問題位於服務層。這讓我理解到 timeout 並不代表整個系統壞掉，而是需要透過多種工具逐層驗證，例如使用 ping 檢查網路連通性、使用 ssh 確認遠端存取、使用 curl 測試應用服務，以及透過 docker 指令檢查服務狀態。這種由下而上的排錯方式，有助於快速定位問題，並提升系統維運與除錯的效率。

