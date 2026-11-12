# GCP 部署流程

本檔是 GCP 正式環境（專案 `plinkopoly`）的實際操作流程。通用的 compose 說明見 [Docker 生產部署](./Docker生產部署.md)，容量與 CPU 配額見 [資源預算](../20-排障與監控/資源預算.md)，這次部署的實測與優化過程見 [週期寫回優化與 GCP 壓測紀錄](../90-參考附件/歷史/週期寫回優化與GCP壓測紀錄-2026-11.md)。

## 1. 架構與資源

| 項目 | 內容 |
|---|---|
| 專案／區域 | `plinkopoly`、asia-east1-b |
| 應用 VM | `pja-game-1`，e2-standard-8，Ubuntu 24.04，使用者 `User`，固定外部 IP `pja-game-ip`（34.81.192.196） |
| 資料庫 | Cloud SQL `pja-mysql`，MySQL 8.4，單區、100GB SSD、自動備份＋PITR、刪除保護；規格 `db-custom-8-32768`（可降為 `db-custom-4-16384`） |
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
- 升降資料庫規格：`gcloud sql instances patch pja-mysql --tier=db-custom-8-32768`（會重啟）。降到 4 vCPU 前先把 `mysql.flush_concurrency` 調回 16、`max_open_conns` 調回 40，否則資料庫 CPU 會頂滿。

## 7. 正式設定現況

| 項目 | 值 |
|---|---|
| World image | `project-world:1.0.6-group` |
| 各服務 `cpus` | World 4.0、Gate 2.0、Central 1.5，其餘 1.0 |
| `mysql.flush_concurrency`／`max_open_conns`／`max_idle_conns` | 32／80／80（Cloud SQL 8 vCPU） |
| `mysql.flush_group_size` | 8（預設） |
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
