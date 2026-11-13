# GCP 部署流程

本檔是 GCP 正式環境（專案 `plinkopoly`）的實際操作流程。通用的 compose 說明見 [Docker 生產部署](./Docker生產部署.md)，容量與 CPU 配額見 [資源預算](../20-排障與監控/資源預算.md)，這次部署的實測與優化過程見 [週期寫回優化與 GCP 壓測紀錄](../90-參考附件/歷史/週期寫回優化與GCP壓測紀錄-2026-11.md)。

## 1. 架構與資源

| 項目 | 內容 |
|---|---|
| 專案／區域 | `plinkopoly`、asia-east1-b |
| 應用 VM | `pja-game-1`，e2-standard-8，Ubuntu 24.04，使用者 `User`，固定外部 IP `pja-game-ip`（34.81.192.196） |
| 資料庫 | 現行：VM 上 compose 的 `mysql` 容器（見「本機 MySQL 與每日備份」）；Cloud SQL `pja-mysql`（MySQL 8.4，單區、100GB SSD、自動備份＋PITR、刪除保護，規格 `db-custom-8-32768`）已停機保留 |
| 資料庫連線 | Private Service Connect，端點 `pja-sql-psc`，私網 IP `10.140.0.7`（VPC 對等在此專案連不起來） |
| Image Registry | `asia-east1-docker.pkg.dev/plinkopoly/pja`，image 名稱 `project-<服務>:<tag>` |
| 同機服務 | Central、World、Gate、Record、Redis、RabbitMQ、Prometheus、Alertmanager、Grafana（docker compose） |
| 暫時關閉 | Chat、Lobby（`compose.slim.yml` 放進 profile，防火牆只開 62001） |
| 對外埠 | 62001（Gate）、8080 |

VM 上的目錄：

| 路徑 | 內容 |
|---|---|
| `/srv/pja/app` | `docker/`、`monitoring/`、`scripts/`、`.env.prod`、`compose.sh` |
| `/srv/pja/config-data` | 遊戲設定 artifact root（`current.json` 與 `releases/`），唯讀掛進 World |
| `/srv/pja/s2s-secrets` | S2S 憑證（目前 S2S 驗證已關） |
| `/srv/pja/backups` | 設定備份（壓測前設定、舊 release、骰子原檔等） |

啟動一律用 `/srv/pja/app/compose.sh`，它固定帶 `.env.prod` 與 `compose.prod.yml`、`compose.cloudsql.yml`、`compose.slim.yml` 三個檔：

```bash
./compose.sh up -d world            # 只換 World
./compose.sh up -d --force-recreate central world gate
```

## 2. 只存在 VM、偏離 repo 的設定

- `docker/prod/config/*.yaml` 的 `s2s_auth.enabled=false`（S2S 驗證已關；要重開改回 `true` 並重啟服務）。
- `gate.yaml` 的 `public_addr` 為 `34.81.192.196:62001`，且移除 `dial.chat`；`prometheus.prod.yml` 移除 chat／lobby job。
- `compose.cloudsql.yml`、`compose.slim.yml` 為 VM 專用覆蓋檔，本機 mysql 容器只在 `local-mysql` profile 啟動（資料卷保留未刪）。
- `.env.prod` 在 VM（gitignored），內含密碼與 `WORLD_IMAGE`、`DATA_CONFIG_ROOT`、`MYSQL_*`、`DEBUG_PPROF_BASIC_AUTH` 等。
- 正式一律 `AUTO_MIGRATE_DISABLED=true`，改結構要手動 SQL。

## 3. 連線與工具

- SSH 走 IAP：`gcloud compute ssh pja-game-1 --zone=asia-east1-b --tunnel-through-iap --project=plinkopoly`。
- Windows 注意：
  - 用 PowerShell 的 `gcloud`；git-bash 的 gcloud 路徑含空格會失敗，`/26` 之類字串會被轉成路徑。
  - `--command` 內有引號、管線或多個字時會壞，改寫成 `.sh`，用 `gcloud compute scp` 傳到 VM 再 `bash` 執行。
  - 含中文的腳本不要用 PowerShell 的 `Get-Content`／`Set-Content` 重寫（會損毀編碼、吃掉換行），用 bash `sed` 改。
- VM 拉 image 靠 VM 服務帳號（已授權 `artifactregistry.reader`）。`docker login` 用 metadata token，約 1 小時失效，`compose pull` 前要重登：

```bash
TOKEN=$(curl -s -H 'Metadata-Flavor: Google' http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')
echo "$TOKEN" | sudo docker login -u oauth2accesstoken --password-stdin asia-east1-docker.pkg.dev
```

## 4. 發版流程（以 World 為例）

### 4.1 本機：建置與推送 image

每個服務有自己的健康檢查埠，建 image 要帶對：World 是 **6062**（其他服務見各自 compose 設定）。

```bash
docker build -f docker/app.Dockerfile --build-arg APP_DIR=cmd/world --build-arg HEALTHCHECK_PORT=6062 --build-arg VERSION=<tag> \
  -t asia-east1-docker.pkg.dev/plinkopoly/pja/project-world:<tag> .
docker push asia-east1-docker.pkg.dev/plinkopoly/pja/project-world:<tag>
```

tag 建議帶遞增序號與主題（例如 `1.0.6-group`），方便回滾時辨識。

### 4.2 遊戲設定 release（World 版本變動時必做檢查）

release 的 `checksums.json` 記錄 `schema_hash`（由編進 World 的 `pkg/dataconfig/generated/*.go` 算出）。新版 World 的 hash 與 VM 上 release 不同時，World 啟動報 `dataconfig: config schema is not supported`，重試不會成功。

1. 在本機以新版原始碼重發 release 到暫存目錄：

   ```powershell
   .\scripts\publish-data-config.ps1 -Source config\data -Root <暫存目錄>
   ```

2. 打包上傳到 VM，先備份舊的整個 `config-data`（例如 `cp -a /srv/pja/config-data /srv/pja/backups/config-data-<舊 schema 簡稱>`），再換掉 `releases/` 與 `current.json`。
3. 回滾到舊 image 時，要一併換回舊 release。

### 4.3 資料庫變更

- 先看這次版本有沒有 DDL（見 [Docker 生產部署](./Docker生產部署.md) 的手動升級 SQL 清單）。例：抽獎唯一索引，部署前先檢查沒有重複種子再 ALTER。
- 用 VM 上的 mysql client 容器連 Cloud SQL（root 密碼在 `.env.prod`）：

  ```bash
  sudo docker run --rm -e MYSQL_PWD="$ROOTPW" mysql:8.4 mysql -h 10.140.0.7 -uroot -e "<SQL>"
  ```

- 大表 DDL 先評估鎖表時間；Cloud SQL 有自動備份與 PITR，變更前可手動建一次備份。

### 4.4 VM：換 image 並驗證

```bash
cd /srv/pja/app
sed -i 's#^WORLD_IMAGE=.*#WORLD_IMAGE=asia-east1-docker.pkg.dev/plinkopoly/pja/project-world:<tag>#' .env.prod
./compose.sh up -d world
sudo docker ps --format '{{.Names}} {{.Status}}'
sudo docker logs --since 2m project-prod-world | grep -E 'error|batch writer enabled|app started'
```

驗證清單：

1. `project-prod-world` 為 `healthy`，日誌沒有 `error`。
2. 日誌有 `snapshot reader enabled`、`batch writer enabled`（寫回單次往返已啟用）。
3. 沒有 `AUTO_MIGRATE_DISABLED=true but indexes are missing` 警告（有就補索引）。
4. Prometheus／Grafana 看 `game_player_flush_failures_total` 不增加、`Player Flush Batch Results` 以 `committed` 為主。

只想改設定（不換 image）：改 `docker/prod/config/*.yaml` 後 `sudo docker restart project-prod-<服務>`（設定在啟動時讀）。

### 4.5 回滾

1. `.env.prod` 的 `WORLD_IMAGE` 改回舊 tag，`./compose.sh up -d world`。
2. 資料設定 release 若換過，換回舊 release。
3. DDL 以加法為主（新索引、新欄位）時舊 image 相容，不必回退；刪除型 DDL 要依備份處理。

## 5. 建置新環境（首次建表）

1. 建 VM、Cloud SQL（私網端點 PSC）、防火牆（只開 62001、8080，SSH 走 IAP）、Artifact Registry。
2. 在 VM 準備 `/srv/pja/app` 與 `.env.prod`，發布遊戲設定 release 到 `/srv/pja/config-data`。
3. 首次建表用 AutoMigrate 覆蓋檔，建完再回正式設定：

   ```bash
   ./compose.sh -f docker/compose/compose.firstrun.yml up -d --force-recreate central world gate record
   # 等到四個服務 healthy 後
   ./compose.sh up -d --force-recreate central world gate record
   ```

4. 手動建立 AutoMigrate 不會建的索引或差異（以 `docs` 的升級 SQL 清單與日誌警告為準）。

## 6. 停機與恢復（省費用）

停機：先停 VM，再停 Cloud SQL。恢復：先開 Cloud SQL，再開 VM；VM 開機後 compose 服務依 `restart: unless-stopped` 自動起來。

```bash
gcloud compute instances stop pja-game-1 --zone=asia-east1-b --project=plinkopoly
gcloud sql instances patch pja-mysql --project=plinkopoly --activation-policy=NEVER
```

```bash
gcloud sql instances patch pja-mysql --project=plinkopoly --activation-policy=ALWAYS
gcloud compute instances start pja-game-1 --zone=asia-east1-b --project=plinkopoly
```

- 停機後仍收費：磁碟、Cloud SQL 儲存、固定外部 IP（閒置時計費，金額小）。
- Cloud SQL 停機時只能改啟用狀態與刪除保護，其他設定（含備份）會被拒絕（`HTTP 400`）；要改設定得連同 `--activation-policy=ALWAYS` 一起送出。
- 停機期間每日自動備份不會執行。

### 備份設定（2026-11-13 查證與調整）

| 項目 | 值 |
|---|---|
| 自動備份 | 啟用，每日 19:00 UTC（台灣 03:00） |
| 備份保留 | 最近 14 份 |
| 時間點還原（PITR） | 啟用（binlog），交易日誌保留 7 天，存 Cloud Storage |
| 刪除保護 | 啟用 |
| 可用性 | 單區（`ZONAL`），無高可用、無異地副本 |

調整指令（實例執行中才可改）：

```bash
gcloud sql instances patch pja-mysql --project=plinkopoly --retained-transaction-log-days=7 --retained-backups-count=14
```

查詢：

```bash
gcloud sql instances describe pja-mysql --project=plinkopoly --format="yaml(settings.backupConfiguration)"
gcloud sql backups list --instance=pja-mysql --project=plinkopoly
```

仍待補：還原演練（還原到暫時實例並量測時間）、長期保留（每週或每月備份）、異地副本、上線前評估高可用。專案的 `scripts/docker-backup-mysql.sh` 只用於 compose 內的 MySQL 容器，不適用 Cloud SQL。
### 本機 MySQL 與每日備份（2026-11-13 起現行做法）

資料庫改在 VM 上用 compose 的 `mysql` 容器跑（資料卷 `project-prod_prod-mysql-data`），Cloud SQL `pja-mysql` 已停機保留（見上）。VM 的 `compose.sh` 不再帶 `compose.cloudsql.yml`，原版留在 `compose.sh.cloudsql`；要改回 Cloud SQL 就換回它並啟動實例。

每日備份流程（`/srv/pja/backup-daily.sh`，VM 使用者 crontab，每天 04:00 UTC＝台灣 12:00）：

1. `scripts/docker-backup-mysql.sh` 用 `mysqldump --single-transaction` 匯出成 `/srv/pja/backups/mysql/mysql-<時間>.sql.gz`，保留最近 14 份。
2. `gzip -t` 驗證壓縮檔、檔案小於 1KB 視為失敗。
3. 上傳到 `gs://plinkopoly-pja-backups/mysql/daily/`（區域 asia-east1，禁止公開、統一桶層級權限；生命週期規則 30 天後刪除）；VM 服務帳號對該桶有 `objectAdmin`。
4. 成功後寫 `/srv/pja/backups/mysql/.last-success`；每次執行的輸出在 `/srv/pja/backups/backup.log`，失敗時腳本非零結束。

MySQL 資料目錄在獨立的 100GB SSD 持久碟 `pja-mysql-data`（掛在 `/srv/pja/mysql-data`，由 `docker/compose/compose.localdisk.yml` 以 bind 卷掛進容器）。**不要把資料放在 pd-standard 開機碟**：2026-11-13 第一次本機資料庫壓測就因此失敗（寫入 IOPS 太低，寫回逾時、斷路器開啟，5000 人寫回只剩約 10 位／秒、失敗計數 86 萬）；換 SSD 碟後同樣負載 0 失敗。

另有 GCP 磁碟快照排程 `pja-daily`（每天 19:00 UTC，保留 7 天）同時掛在開機碟與資料碟 `pja-mysql-data` 上，等於整機備份。快照只是當機一致，不取代邏輯備份。

本機資料庫壓測結果（2026-11-13，e2-standard-8 全部同機、MySQL 容器預設 `innodb-buffer-pool-size=256M`、`flush_concurrency` 16、連線池 40）：5000 人登入 0 失敗；擲骰 p50 約 10 毫秒、p99 約 75 毫秒（Cloud SQL 8 vCPU 時約 3／50 毫秒）；寫回約 320 位／秒、每輪髒玩家平均約 1400、寫回失敗 0；MySQL 約 2.5 核、World 約 2.0 核、Gate 約 0.64 核，主機 load 約 6／8。機器人的操作頻率遠高於真人，真實玩家的寫回負載會低很多。

**MySQL 緩衝池 8GB 重測（同一環境，5000 人）沒有改善**：寫回約 270 位／秒（256MB 時約 320）、擲骰 p50／p99 約 30／200 毫秒（256MB 時約 10／75，差異在雜訊內）、MySQL 仍約 2.2 核。資料量小、本來就放得進 256MB，瓶頸在寫入（redo／fsync），不在讀取，所以不必調大；`compose.prod.yml` 的 `--innodb-buffer-pool-size=256M` 維持原樣。

**各服務記憶體上限測試**（2026-11-13，e2-standard-8 全部同機，預設上限：Central 640m、World 640m／Go 軟上限 512MB、Gate 480m／軟上限 384MB）：

| 人數 | World RSS | Gate RSS | 結果 |
|---|---|---|---|
| 5000 | 約 315 MB | 約 173 MB | 擲骰 p50 約 3～10 毫秒，穩定 |
| 10000 | 約 524 MB（堆積 475MB，貼著軟上限） | 約 365 MB（堆積 312MB，貼著軟上限） | 0 失敗、沒有 OOM，但 GC 吃掉 CPU：Gate 2.15 核（5000 人時 0.67）、擲骰 p50 約 600 毫秒、p99 約 1.5 秒 |
| 10000，World 上限調 1024MB（mem 1280m）、Gate 調 768MB（mem 960m） | 約 828 MB（堆積 760MB） | 約 432 MB | Gate 0.85 核、擲骰 p50 約 75 毫秒、p99 約 400～600 毫秒；客戶端 CPU 平均 18%（不是瓶頸） |

結論：
- **預設上限撐 5000 人綽綽有餘；到 10000 人會因 Go 軟上限觸發頻繁 GC 而變慢（不會 OOM）**。World 每位玩家約 50～80KB 堆積，Gate 每條連線約 40KB。
- 調整公式（含 GC 餘裕）：World `go.memory_limit_mb` ≈ 100 ＋ 0.1 × 人數（MB），`mem_limit` 取 ÷0.8；Gate `go.memory_limit_mb` ≈ 40 ＋ 0.07 × 人數，`mem_limit` 取 ÷0.8。例如 10000 人：World 1024／1280m、Gate 768／960m（compose 與 yaml 要同步改，`configaudit` 會比對）。
- 觀察：5000 與 10000 人的總擲骰量都落在約 3100～3300 次／秒（10000 人時每人只剩約一半頻率），疑似 World 的處理量上限；原因沒有查（World CPU 約 2.5／4 核未飽和）。機器人每人每秒擲骰比真人頻繁很多，真人不會到這個量，但要估更高人數時要注意這個數字。

這些是壓測用的暫時調整，已還原；repo 的預設值沒有改，要上到 10000 人前再依上面公式調整。

### 浸泡測試（2026-11-13，6 小時、5000 人）

條件：單台 e2-standard-8（應用與本機 MySQL 同機，pd-ssd 資料碟）、客戶端另一台 e2-standard-4；5000 名機器人每人約每 1.5 秒擲骰（總計約 3300 次／秒）；Gate 與 World 預設記憶體上限；MySQL 緩衝池 256MB。

| 項目 | 結果 |
|---|---|
| 登入／斷線 | 5000 全數登入，0 次失敗、0 次斷線重連 |
| 擲骰 | 共 7227 萬次，0 次逾時遺失；p50 5～24 毫秒（每 5 分鐘取樣，多數 6～12）、p99 80～150 毫秒（偶有 440 毫秒尖峰）、最大值偶爾 0.5～1.5 秒 |
| 記憶體 | 全程平穩：World 約 317～332 MB、Gate 約 260 MB、Central 約 25 MB、MySQL 約 945 MB，無增長趨勢，沒有 OOM 或重啟 |
| CPU | World 約 2.3～2.8 核、MySQL 約 2.0～2.6 核、Gate 約 0.7～0.8 核；客戶端平均 11% |
| 錯誤日誌 | World 只有 `gorm slow query`（約 750 筆，發生時段沒有逐一確認）與見下的 `gorm query failed`；Central 有新帳號限速與審計慢查詢（登入階段） |

發現與待辦：
- **MySQL binlog 長得非常快**：5000 人持續寫入時約 16 GB／小時（每個檔 1.1GB、4 分鐘一個），預設保留 30 天會在幾小時內塞滿 98GB 資料碟。壓測時暫時設 `binlog_expire_logs_seconds=1200`，測完已還原。正式環境真實寫入量遠低於機器人，但**仍須明確設定保留天數**（例如 3 天），並決定是否把 binlog 上傳 GCS 做時間點還原。
- **資料量成長**：6 小時資料目錄從 6GB 長到 47GB，最大的表依序是 `player_mission_event`（約 26GB）、`player_lottery_draw`（約 7.5GB）、`record_outbox_event`（約 6.8GB）、`record_events_<日期>`（約 5.8GB）。機器人行為不等於真人，但這幾張表需要保留／清理策略與容量估算（尤其任務事件與 outbox）。
- **`gorm query failed` 假警報**：World 日誌出現上千筆（已被取樣抑制），錯誤訊息是 `mysqlstore: save step is not recordable`。這是批次寫回遇到需要先讀取的步驟（移除星星要先 `SELECT player_board_road_star`）時刻意回傳的哨兵錯誤，呼叫端會退回逐句寫回，功能正常，但 GORM 日誌把它記成 warn。建議在 `pkg/mysql/gorm_logger.go` 的錯誤過濾忽略它（尚未修）。
- **登入階段限速**：5000 人在約 10 分鐘內登入完（Central 新帳號限速、Gate 單 IP 限速），Central 記到 46 筆取樣的 `new user rate limited`，客戶端重試後全部成功。
- **骰子帳目**：結束時 5000 位玩家骰子總和 9,491,858，但這個環境先前有一次被中斷的 1024 人試跑也消耗過骰子，所以無法用「初始 − 擲骰次數」精確核對；本次以 0 次逾時、0 次斷線、World 無寫回失敗日誌作為「沒有資料遺失」的依據，不是精確帳目。
- 為了讓 5000 人跑得起來，VM 暫時改了：Gate `max_connections` 6000、`password_enabled` true、`max_logged_in_per_ip` 200，以及放大骰子；**正式範本的 `max_connections` 是 1024，只收得下約 1000 位玩家，上線前要依預期人數調整**。測完全部已還原並清庫。

2026-11-13 實測（還原演練）：寫入 5000 筆探測資料 → 備份 → 上傳 → 從 GCS 下載（雜湊與本機相同）→ 還原到暫時資料庫：筆數 5000、校驗碼、64 張表全部一致，測完已清理。

限制與待補：
- 時間點還原：MySQL 8.4 預設開 binlog（本機保留 30 天），但沒有同步到 GCS；VM 與磁碟一起壞掉時只能回到最近一次每日備份。
- 沒有備份失敗告警（只有 `.last-success` 時間戳與日誌）；可用它超過 26 小時沒更新當告警條件。
- 單機單磁碟：每日備份是唯一的異地副本，要定期（每月）實際還原一次。
- 手動即時備份：`/srv/pja/backup-daily.sh`；還原用 `scripts/docker-restore-mysql.sh`（會覆蓋正式庫，先停應用）。

- 升降資料庫規格：`gcloud sql instances patch pja-mysql --tier=db-custom-8-32768`（會重啟）。降到 4 vCPU 前先把 `mysql.flush_concurrency` 調回 16、`max_open_conns` 調回 40，否則資料庫 CPU 會頂滿。

## 7. 正式設定現況

| 項目 | 值 |
|---|---|
| World image | `project-world:1.0.6-group` |
| 各服務 `cpus` | World 4.0、Gate 2.0、Central 1.5，其餘 1.0 |
| `mysql.flush_concurrency`／`max_open_conns`／`max_idle_conns` | 32／80／80（Cloud SQL 8 vCPU） |
| `gate.session.max_logged_in_per_ip` | 50（上線後看 `tcp_rejected_connections_total{reason="logged_in_per_ip"}` 再調） |

## 8. 壓測前後的暫時調整

壓測要暫時放寬 VM 設定，原檔先備份到 `/srv/pja/backups/cfg-pre-loadtest`，測完全部還原並清庫：

- `gate.yaml`：`max_connections` 5500、`memory_limit_mb` 1024、`password_enabled` true、`session.max_logged_in_per_ip` 100。
- `world.yaml`：`max_players` 6000。`central.yaml`：新帳號速率 `rate` 100、`burst` 500。
- 暫時的 compose 覆蓋檔（Gate `mem_limit: 1280m`，各服務 `cpus` 放寬），測完刪除。
- 持續寫回負載要放大遊戲設定的初始骰子數與上限，否則機器人約 7 分鐘後骰子用完、不再產生寫回。做法與腳本思路見壓測紀錄的第 5 節。
- 還原：停四個服務 → 還原設定與 release → 清庫（`DROP DATABASE`＋`CREATE DATABASE`）與 Redis `FLUSHALL` → 用首次建表覆蓋檔重建表 → 回正式設定。

## 9. 常見問題

| 現象 | 原因與處置 |
|---|---|
| World 重啟迴圈，`config schema is not supported` | release 的 schema hash 與新版不符；見 4.2 |
| World 啟動時 `missing: …ux_…` 索引警告 | 正式不跑 AutoMigrate，補 DDL（見 4.3） |
| `compose.*.yml` 報 `additional properties … not allowed` | 覆蓋檔被損毀（常因 PowerShell 重寫中文），用 `cat -A` 檢查，註解行與 `services:` 是否黏在一起 |
| `compose pull` 未授權 | metadata token 過期，重做第 3 節的 `docker login` |
| ssh `--command` 內 `pkill -f 名稱` 沒作用 | 會殺掉自己的 shell，寫成 `pkill -f '[n]ame'` |
| 子模組指標在遠端找不到 | 先 push 子模組，再 push 主倉庫 |
