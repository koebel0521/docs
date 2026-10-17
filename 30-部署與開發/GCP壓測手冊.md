# GCP 壓測手冊

在 Google Compute Engine 上用 VM 跑 5000 人壓測的規格、步驟與判讀。本機（單機 loopback）壓測的基準見[容量與壓測基準](../20-排障與監控/容量與壓測基準.md)，工具與食譜見[壓測手冊](壓測手冊.md)。

> 腳本在 `tools/private/gcp-loadtest/`（不進主倉庫版控）。**目前只做了語法與 `docker compose config` 檢查，沒有實際對 GCP 執行過**；第一次使用請先跑方案 0，並以實際結果修正本頁。

## 規格依據

來自本機 1500～3800 人的實測（逐步數字見[壓測紀錄第四輪](../90-參考附件/歷史/壓測紀錄-2026-10-16-第四輪.md)）。World、Record、MySQL 的 CPU 沒有單獨量到，所以規格留餘裕，跑完方案 1 後看實際用量再收斂。

| 元件 | 5000 人估計 | 依據 |
|---|---|---|
| Gate | 記憶體約 0.8～1GB、CPU 約 0.4 核 | 3800 連線約 600MB（每連線約 157KB）、0.3 核；每人每 3 秒擲骰時 5000 人約 1700 次／秒 |
| World | 記憶體約 250MB | 3800 人約 184MB |
| Central | 約 70MB | 登入尖峰才吃 CPU（bcrypt） |
| MySQL | 緩衝池至少 2～4GB | dev 預設只有 128MB，資料大時冷讀會慢（第 35 步） |
| Redis | 低於 100MB | 但延遲最敏感，必須與 World 同可用區（第 28、36 步） |

## 必須調整的限制

`gen-config.sh` 會把 `config/docker/*.yaml` 複製到 `.out/config` 並套用，每一處都用 `sed` 取代並確認恰好命中一處（上游改檔後會直接失敗而不是悄悄失效）：

| 設定 | 預設 | 5000 人 | 原因 |
|---|---|---|---|
| Gate `max_connections` | 1024 | 5500 | 否則第 1025 個連線被拒 |
| Gate `go.memory_limit_mb` | 512 | 1280 | 5000 連線約 0.8～1GB；上限低於實際用量時延遲明顯變差（第 39 步） |
| World `max_players` | 5000 | 6000 | 剛好 5000 會把第 5001 位擋掉 |
| World `go.memory_limit_mb` | 512 | 768 | 留餘裕 |
| MySQL `innodb_buffer_pool_size` | 128MB（dev） | 2G（遊戲庫）、512M（Central 庫） | 避免冷讀 |

**單一來源 IP 最多 100 條連線**是寫死在程式裡的常數（`MaxConnectionsPerIP`），不是設定；另有每來源 IP 接入 10 次／秒、登入 30 次／10 秒的限流。5000 人至少要 50 個來源位址。做法：客戶端 VM 配 GCP **別名 IP 範圍**（`/26` 共 64 個位址，可承載約 6000 條連線），`soakload -srcnet <網段>` 讓玩家輪流綁定其中的位址。客戶端必須直連 Gate 的內網 IP；中間放負載平衡器時 Gate 看到的來源都是同一個位址，會被單 IP 限流擋住。

## 方案與估價

| 方案 | 機器 | 估價（每小時） | 用途 |
|---|---|---|---|
| 0 冒煙 | 伺服端 `e2-standard-4`（全部同機）＋ 客戶端 `e2-standard-2`（`/27` 別名範圍） | 約 $0.23 | 300～1000 人、約 1 小時，驗證部署 |
| 1 最小 5000 人 | 伺服端 `e2-standard-8`（全部同機）＋ 客戶端 `e2-standard-4`（`/26`） | 約 $0.47 | 先看 5000 人能不能進、延遲如何 |
| 2 建議 5000 人 | 應用 VM `e2-standard-8` ＋ 資料 VM `e2-standard-4`（MySQL×2、Redis、RabbitMQ，SSD 200GB）＋ 客戶端 `e2-standard-4`（`/26`） | 約 $0.62 | 接近正式的邊界，能看到網路往返的影響 |

**價格是估算**：以第三方價格表的 `e2-standard-2` 在 asia-east1 約 $0.0776／小時按 vCPU 線性放大，沒有核對官方牌價；確切金額請查 [Compute Engine 價格頁](https://cloud.google.com/compute/vm-instance-pricing)（選 asia-east1）。測 3～4 小時約數美元。客戶端用 Spot（被回收就重跑）；伺服端不要用 Spot。

方案 2 共 16 vCPU，新專案的區域 CPU 配額可能只有 8～12 核，要先申請提高。

## 步驟

前置：`gcloud` 已登入並有專案的 Compute 管理權限；本機有 Go（交叉編譯 `soakload`）與 `git`。

```bash
cd tools/private/gcp-loadtest
export PROJECT=<專案 ID> PLAN=0        # 先跑方案 0
./create.sh                            # 防火牆（僅 IAP SSH 與子網內互連）與 VM
./deploy.sh                            # 送出 HEAD 原始碼、啟動 compose、送 soakload
# 等 gate 容器 healthy（ssh 進應用 VM：sudo docker ps）
N=500 HOLD=10m PASSWORD='<自訂密碼>' ./run-test.sh
```

方案 1、2 把 `PLAN` 改成 1、2，`N=5000`。結束後：

```bash
./teardown.sh                          # 刪 VM 與防火牆規則，磁碟隨 VM 刪除
```

## 怎麼判讀

- 看 Grafana（應用 VM 的 `127.0.0.1:3000`，用 `gcloud compute ssh --tunnel-through-iap -- -L 3000:127.0.0.1:3000` 轉出）與 Prometheus 告警，不依賴客戶端的取樣欄位：方案 0、1 沿用 `compose.full.yml`，診斷埠只綁 loopback，客戶端取不到（`soakload` 的伺服端欄位會是 0）；方案 2 的應用 VM 把診斷埠開在內網，客戶端取樣可用。
- 登入階段：新帳號要 bcrypt，吞吐受 `max(2, CPU 數 ÷ 2)` 個並行位限制；5000 位新帳號進場要幾分鐘到十幾分鐘，`run-test.sh` 用 `-pace 100ms` 配合單來源位址登入速率。
- 通過標準（本機基準，GCP 上應相同或更好）：5000 連線全部在線；擲骰 p99 在個位數 ms～十幾 ms；Gate 堆積的 goroutine 約 3／連線；無 `GameQueueOverflow`、`GameLoginCentralBusy` 長時間 firing。
- 要找上限：把 `EVERY` 從 `3s` 降到 `1s`、`300ms`，看哪個元件先飽和（Gate CPU、World、MySQL、Redis）。

## 已知風險

- **腳本沒實測**：`aliases=/26` 的 `--network-interface` 語法、別名 IP 在客戶端是否可直接綁定（依賴 GCP guest agent 設定本機路由）、IAP SSH 都未驗證。方案 0 就是用來驗證這些。
- 伺服端 VM 有外部臨時 IP（為了拉映像），入口由防火牆限制為 IAP；要更嚴格可改 Cloud NAT 並移除外部位址。
- Gate 被 OOM kill 或容器重啟時所有玩家掉線（第 40 步）；compose 已設 `restart: unless-stopped`。
- S2S 認證在 `config/docker` 預設是關閉的，這份壓測沒有開；開啟的成本見壓測紀錄第 30、37 步。
- 防火牆開 `tcp:6060-6066`（診斷埠）給子網內，因為 `DEBUG_ALLOW_CIDRS` 預設允許 10.0.0.0/8；若 VM 子網不在這些範圍，診斷埠會拒絕。
