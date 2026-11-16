# CDN 與靜態檔案發布

本檔說明遊戲設定檔與客戶端資源包如何放到 CDN、誰負責哪一段，以及 GCP 上的建置與發布步驟。狀態：**方案，尚未建置**；建置前先確認文末「待決定事項」。

CDN 只放靜態檔案。遊戲連線走 Gate 的 TCP 62001，不經過 CDN。

## 1. 分工

| 工作 | 負責 |
|---|---|
| 建儲存桶與 CDN、買網域、設 DNS 與 HTTPS | 營運（GCP 帳號） |
| 發布：新版設定檔、資源包上傳 | PJA-Config 與營運 |
| 下載邏輯：啟動時抓 `latest.json`、比對雜湊、只下載有變的檔案、失敗退回內建版本 | PJA-Client（不在本專案範圍） |
| 資源包的打包與使用方式 | PJA-Client |
| Server | 幾乎不用做，只保留兩件事（見第 2 節） |

設定檔的檔案結構、`latest.json` 與 `manifest.json` 欄位、Client 下載流程，已在[版本相容重新規劃](../90-參考附件/方案草稿/版本相容重新規劃.md)第 6.5 節定案，本檔不重複。

## 2. Server 要維持的事

1. **`min_config_seq` 一致**：發新版設定檔時，Server 的 `pkg/release/release.json` 內的 `config_seq`、`min_config_seq`、`client_contract`，要和 CDN 上 `latest.json` 的同名欄位一致；不一致時，Client 下載到的版本與 Server 判斷的「太舊」不同步。
2. **CDN 故障不影響 Server**：Server 的 `ConfigTooOld` 判斷只依 Client 連線時回報的 `configSeq`，不依賴 CDN；CDN 掛掉時，玩家用內建設定照常連線。

## 3. 網址

- 用自己網域下的子網域，例如 `cdn.<網域>`，不要用 `storage.googleapis.com/<桶>/…`。原因：客戶端會寫死網址，已發出的 App 無法改；用自己的網域，之後換後端（Google CDN、Cloudflare 等）只改 DNS，App 不用更新。
- 不要用「桶名當網域＋CNAME 到 `c.storage.googleapis.com`」：只有 HTTP，iOS 與 Android 預設擋 HTTP。
- 設定檔與資源包用路徑分開：`/config/…`、`/assets/…`。資源包流量大到需要單獨換服務時，再拆成 `assets.<網域>`。

## 4. 路徑與快取規則

| 路徑 | 內容 | `Cache-Control` |
|---|---|---|
| `/config/latest.json` | 指標檔，說明目前最新版 | `public, max-age=60` |
| `/config/<config_version>/…` | 該版設定檔與 `manifest.json`（不可變） | `public, max-age=31536000, immutable` |
| `/assets/<版本>/<平台>/<檔名>` | 資源包，iOS 與 Android 分開（不可變） | `public, max-age=31536000, immutable` |

規則：
- 版本路徑的檔案上傳後不再修改，改內容就換新版本路徑；因此不需要主動清除快取。
- 只有 `latest.json` 會被覆蓋，靠 60 秒到期；緊急回滾就是把 `latest.json` 改回舊版。
- 舊版本保留至少 30～90 天，舊 App 還會用舊路徑，不要發完新版就刪。

## 5. 方案選擇

| 方案 | 固定費用 | 適用 |
|---|---|---|
| A. Cloud Storage 公開網址直連（無 CDN） | 無 | 只有小設定檔、玩家集中在台灣或亞洲；但用 `storage.googleapis.com` 網址，之後換後端要更新 App |
| B. Cloud Storage + 外部 Application Load Balancer + Cloud CDN | 轉發規則約每月 US$18（約 NT$600） | 有資源包、玩家多或有海外玩家、要 Cloud Armor 與 GCP 帳單集中 |
| C. Cloudflare（或 R2）放在 GCS 前面 | 低 | 資源包大、流量大、要壓低流量費；免費版用於大量資源包的條款要先確認 |

建議：現在只有小設定檔時，網址一律先定成自己網域下的子網域；後端先走 A 或 C，資源包定案、流量上來再升到 B。價格都是粗估，建置前用官方計算機核對。

流量費估算：每位玩家下載量 × 玩家數 × 單價。例如首次下載 100 MB、每月新增 1 萬人，約 1 TB；Google CDN 亞洲約 US$0.04～0.09／GB。

## 6. 建置步驟（方案 B）

假設網域為 `cdn.example.com`，區域 `asia-east1`。

```bash
# 1) 儲存桶（不開公開存取，只讓負載平衡器讀取）
gcloud storage buckets create gs://plinkopoly-cdn --location=asia-east1 --uniform-bucket-level-access

# 2) 後端桶與 CDN：完全依檔案的 Cache-Control 決定快取
gcloud compute backend-buckets create pja-cdn-bucket --gcs-bucket-name=plinkopoly-cdn \
  --enable-cdn --cache-mode=USE_ORIGIN_HEADERS

# 3) 靜態 IP、網址對應、託管憑證、HTTPS 代理與轉發規則
gcloud compute addresses create pja-cdn-ip --global
gcloud compute url-maps create pja-cdn-map --default-backend-bucket=pja-cdn-bucket
gcloud compute ssl-certificates create pja-cdn-cert --domains=cdn.example.com --global
gcloud compute target-https-proxies create pja-cdn-proxy --url-map=pja-cdn-map --ssl-certificates=pja-cdn-cert
gcloud compute forwarding-rules create pja-cdn-fr --global --address=pja-cdn-ip \
  --target-https-proxy=pja-cdn-proxy --ports=443
```

之後：
1. 把 `cdn.example.com` 的 A 記錄指向 `pja-cdn-ip`；託管憑證要等 DNS 生效後簽發，約數十分鐘。
2. 讓負載平衡器能讀儲存桶：依官方文件給 Cloud CDN 的服務帳號桶層級 objectViewer 角色（帳號名稱建置時以官方文件核對）。
3. 驗證：`curl -I https://cdn.example.com/config/latest.json`，確認 200、`cache-control` 為上表的值，重複請求後出現 `age` 標頭代表命中快取。

## 7. 發布流程（設定檔）

1. 設定檔產生新版 `/config/<config_version>/`（含 `manifest.json`）。
2. **先**上傳整個版本目錄，**最後才**覆蓋 `latest.json`，避免玩家讀到半套：

```bash
gcloud storage cp -r config/<config_version> gs://plinkopoly-cdn/config/ \
  --cache-control="public, max-age=31536000, immutable"
gcloud storage cp latest.json gs://plinkopoly-cdn/config/ \
  --cache-control="public, max-age=60"
```

3. 確認 `curl https://cdn.<網域>/config/latest.json` 是新版（最多等 60 秒快取到期）。
4. 對齊 Server：確認 `release.json` 的 `config_seq`、`min_config_seq`、`client_contract` 與 `latest.json` 一致，再依[GCP 部署流程](./GCP部署流程.md)發版。
5. 回滾：把 `latest.json` 改回上一版再上傳；版本目錄不刪。

資源包的發布步驟相同（版本目錄＋最後才更新指標檔）；清單格式由 PJA-Client 定案後補在本檔。

## 8. 待決定事項

1. 網域：用哪個、誰管 DNS（決定第 3 節的實際網址）。
2. 資源包大小（單檔與總量）與預計玩家數（決定選方案 A、B 或 C）。
3. 資源包的清單與下載格式：由 PJA-Client 負責人定案。
4. 是否需要 Cloud Armor 限速或防盜連（資源包有付費內容時才需要）。
5. 儲存桶與 CDN 的 GCP 資源是否另開專案或沿用 `plinkopoly`。
