# Docker 發版流程

這份文件定義目前專案的 release / version 流程，目標是讓：

- image tag 可追蹤
- deploy 有固定順序
- rollback 有明確依據

## 1. 版本號規則

建議至少採用固定版本格式，不要直接用 `latest`。

可接受的格式例如：

- `1.0.0`
- `1.0.1`
- `2026.03.24-1`

原則：

1. 同一次 release，`central / world / gate / chat / lobby / record` 六個服務使用同一個 tag
2. production 不直接部署 `latest`
3. rollback 時回到上一個已知穩定 tag

## 2. release 產物

每次 release 至少應有：

- `project-central:<tag>`
- `project-world:<tag>`
- `project-gate:<tag>`
- `project-chat:<tag>`
- `project-lobby:<tag>`
- `project-record:<tag>`
- 對應 commit id
- deploy 時使用的 `.env.prod` 版本紀錄
- `pkg/release/release.json`（發版名稱、server commit、協定版號與 schema 指紋、設定版本與序號、設定契約、database schema；建置時嵌入每個服務，格式見 [版本握手與強制更新](../10-核心流程/版本握手與強制更新.md)）
- migration plan checksum 與 schema migration 狀態
- 服務 `-version`：第一行只印版本，其後以 `key=value` 印 `release`、`server_commit`、`protocol_version`、`config_version`、`go_version`，可用來核對容器內嵌的發版資訊

## 3. 建議 release 步驟

### 手動步驟

1. 決定版本號；`go run ./tools/pja releasemanifest generate -release <tag> -commit <完整 40 碼 SHA>` 依實際產物寫出 `release.json`
2. `go run ./tools/pja releasemanifest verify` 驗證 server/config/schema/protocol 對應；需先 `git submodule update --init`（migration 目錄不存在時 `generate`／`verify` 都會報錯，不會當成沒有 schema）；`config/data/version.json` 以 `dataconfig.LoadManifest` 載入並驗證（40 字元 SHA、`luban_version`、`config_version` 等必填），`{}` 或缺欄位直接報錯，不再因空檔略過比對；`release.json` 以原子寫入，中斷不留截斷檔
3. build image
4. push image
5. 完成 MySQL backup / snapshot
6. 發布設定 artifact：`.\scripts\publish-data-config.ps1 -Source config\data -Root <DATA_CONFIG_ROOT>`（先加 `-DryRun` 預檢；等同 `go run ./tools/pja configartifact -source config/data -root <DATA_CONFIG_ROOT>`）。先記下發布前 `current.json` 的 `config_version`，回滾要切回它。World 啟動以 `pkg/release/verify.go` 比對映像內嵌的 `config_version`／`config_seq` 與 `current.json` 指向的 artifact，不符拒絕啟動，所以設定與映像必須同版
7. 執行一次性的 `deps/PJA-Database/scripts/migrate-prod.sh`
8. 確認 migration 成功後更新 deploy 主機上的 `.env.prod`
9. 執行 production deploy：同一次發版的所有服務整批替換，不可新舊發版混跑（服務之間的 `ServiceHello` 比對發版身分，不一致會斷線，見 [版本握手與強制更新](../10-核心流程/版本握手與強制更新.md#服務之間)）
10. 跑 production smoke test
11. 若失敗則 rollback server image **並把 `current.json` 切回上一版 `config_version`**（見第 7 節），保留已向前的 database schema

以上順序即 `scripts/ci-deploy.sh` 的 CD 流程（備份、預檢、關服、再備份、發布設定、migration、換映像、等健康）；手動升版時不可漏掉發布設定這步。

### 使用 helper script

可直接執行：

```bash
./scripts/docker-release.sh 1.0.0 registry.example.com/project
```

或直接走維護者本機的統一入口（`scripts/private/` 不進版控，其他人以上面的公開腳本為準）：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerrelease -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
```

若只是先產審批 stamp，也可以先跑：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode changeapproval -ChangeApprovalChangeType release -ChangeApprovalTarget 1.0.0 -ChangeApprovalOwner alice
```

這個腳本會：

1. 設定 `IMAGE_TAG`
2. 檢查升級窗口鎖與變更審批戳記；工作樹有未提交修改時拒絕發版（映像與 `release.json` 記的 `HEAD` 會對不上）
3. 以 `go run ./tools/pja releasemanifest generate`／`verify -config-repo deps/PJA-Config` 依實際產物寫出並驗證 `pkg/release/release.json`（嵌入映像；腳本結束後還原該檔）；`config_version` 不可追溯到 `deps/PJA-Config` 時失敗
4. 執行 [docker-build-images.sh](../../scripts/docker-build-images.sh)
5. 推送前以 `trivy image` 掃描六個映像，有可修補的 HIGH／CRITICAL 漏洞即失敗
6. 執行 [docker-push-images.sh](../../scripts/docker-push-images.sh)

若只想分開跑，也可以直接走維護者本機的統一入口（`scripts/private/` 不進版控，其他人以上面的公開腳本為準）：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerbuild -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerpush -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
```

## 4. deploy 主機更新內容

release 完成後，deploy 主機至少要更新：

- `.env.prod` 內的 `IMAGE_TAG`（compose 不讀它；只有 `scripts/ci-deploy.sh` 用來記錄上一版，手動部署以下列六個 `*_IMAGE` 為準）
- `.env.prod` 內的：
  - `CENTRAL_IMAGE`
  - `WORLD_IMAGE`
  - `GATE_IMAGE`
  - `CHAT_IMAGE`
  - `LOBBY_IMAGE`
  - `RECORD_IMAGE`

例如：

```text
IMAGE_TAG=1.0.0
CENTRAL_IMAGE=registry.example.com/project/project-central:1.0.0
WORLD_IMAGE=registry.example.com/project/project-world:1.0.0
GATE_IMAGE=registry.example.com/project/project-gate:1.0.0
CHAT_IMAGE=registry.example.com/project/project-chat:1.0.0
LOBBY_IMAGE=registry.example.com/project/project-lobby:1.0.0
RECORD_IMAGE=registry.example.com/project/project-record:1.0.0
```

同時建議把下列 trace 一起留存：

- build hash
- commit hash
- config hash
- manifest path/hash
- release 時間

`scripts/docker-release.sh` 會把這些資訊與 releasemanifest 驗證結果寫到 `.run/release/<tag>.txt`，並留一份 `.run/release/<tag>.release.json`。

## 5. deploy 指令

在 deploy 主機的倉庫根目錄執行（與 `scripts/ci-deploy.sh` 相同）；compose 檔內的掛載路徑相對於 `docker/compose/`，不可把 `compose.prod.yml` 複製到別處單獨使用。

```bash
docker compose --env-file .env.prod -f docker/compose/compose.prod.yml up -d
```

若有 monitoring：

```bash
docker compose --env-file .env.prod -f docker/compose/compose.prod.yml --profile monitoring up -d
```

## 6. deploy 後驗證

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/production-smoke.ps1
```

若有 monitoring：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/production-smoke.ps1 -CheckMonitoring
```

## 7. rollback 規則

若 deploy 後 smoke test 失敗或服務異常：

1. 停服務（`docker compose ... stop central world gate chat lobby record`）
2. 把 `current.json` 切回升版前記下的 `config_version`：`.\scripts\rollback-data-config-release.ps1 -Root <DATA_CONFIG_ROOT> -Version <40 碼 SHA>`（等同 `go run ./tools/pja configartifact -root <DATA_CONFIG_ROOT> -switch-version <SHA>`）。舊映像內嵌舊版 `config_version`／`config_seq`，`current.json` 仍指向新 artifact 時 World 會拒絕啟動（`pkg/release/verify.go`），回滾本身會失敗
3. 將 `.env.prod` 的六個 `*_IMAGE` 改回上一版（compose 不讀 `IMAGE_TAG`）
4. 重新執行 deploy
5. 不要對 production 執行 migration rollback
6. 若資料也出問題，依 [[Docker備份還原]] 從已驗證的 backup/snapshot restore，並先隔離流量

rollback 範例（`IMAGE_TAG` 僅作紀錄，實際生效的是六個 `*_IMAGE`）：

```text
IMAGE_TAG=0.9.9
CENTRAL_IMAGE=registry.example.com/project/project-central:0.9.9
WORLD_IMAGE=registry.example.com/project/project-world:0.9.9
GATE_IMAGE=registry.example.com/project/project-gate:0.9.9
CHAT_IMAGE=registry.example.com/project/project-chat:0.9.9
LOBBY_IMAGE=registry.example.com/project/project-lobby:0.9.9
RECORD_IMAGE=registry.example.com/project/project-record:0.9.9
```

然後再跑：

```bash
docker compose --env-file .env.prod -f docker/compose/compose.prod.yml up -d
```

## 8. release 紀錄建議

每次 release 建議至少記錄：

- release tag
- build hash
- config hash
- manifest hash
- database schema version
- migration plan checksum
- git commit
- build 時間
- deploy 主機
- deploy 結果
- rollback 與否

最簡單可以先放在：

- Git tag
- release note
- deploy log

## 9. 不建議的做法

- 直接部署 `latest`
- 不記錄 commit 與 image tag 對應
- deploy 後不跑 smoke test
- rollback 只憑印象，不憑版本

## 10. GitLab 自動發版（CD）

推 `vX.Y.Z` 標籤後，`.gitlab-ci.yml` 依序執行。發版相關 job 的 rules 要求 `$CI_COMMIT_REF_PROTECTED == "true"`，**標籤必須設為 Protected tag**，否則不會觸發發版；`build-images` 與 deploy job 不自動重試（`retry: 0`）。

1. `.release-needs` 的檢查全部通過（含 `alert-rules`、`alloc-guard`），`release-info` 產出 `release.json`；`releasemanifest verify -config-repo deps/PJA-Config` 另要求 `config_version` 是 PJA-Config 子模組指標或其祖先，生成物與子模組漂移時在這裡失敗。
2. `build-images`：建置並推送六個映像（tag 為去掉 `v` 的版本號），各映像 digest 存成 artifact `images.env`。
3. `deploy-staging`：預留。目前無第二台主機，設定 `STAGING_ENABLED=true` 與 `STAGING_ENV_FILE` 才啟用。
4. `deploy-production`：手動按下後執行 [ci-deploy.sh](../../scripts/ci-deploy.sh)。
   `build-images` 與 `deploy-production` 的 `after_script` 結束後都會 `docker logout`，registry 憑證不留在 runner 使用者的 `~/.docker/config.json`。

`ci-deploy.sh` 預期關服維護，流程固定：

1. 先拉新映像，備份 MySQL，並預熱（prewarm）停機期間要執行的 Go 工具：`tools/pja` 編成二進位放進具名 volume `pja-bin`，之後 `configartifact` 直接執行它、不再 `go run`（`go run` 的容器 PID 1 是 go 指令，不會轉送 SIGTERM，中斷時工具收不到訊號）；`deps/PJA-Database` 的 `datamigrate` 同樣預編進 `pja-bin`（`/pja-bin/datamigrate`），停機後的 DB migration 直接執行它、不經 `go run`，不必冷編譯，編譯失敗也在關服之前就中止；所有 `go_run` 容器加 `--init`（PID 1 為 tini，取消部署的 SIGTERM 會轉送給工具，不留孤兒行程）。手動執行的 `deps/PJA-Database/scripts/migrate-prod.sh` 仍走 `go run`。接著以唯讀掛載 `DATA_CONFIG_ROOT` 跑 `configartifact -dry-run` 預檢設定 artifact：來源驗證或換版規則失敗時在這裡中止，**服務尚未停止**。`.env.prod` 以 `scripts/lib/envfile.sh` 的 `env_get` 逐鍵讀（compose dotenv 規則，不 `source`，`MYSQL_DSN` 可不加引號）；CI 中 `ENV_FILE` 必須是絕對路徑。
1a. **關服之前**的 preflight：`preflight_config` 之後再跑 `preflight_secrets`，檢查 `.env.prod` 的 `REDIS_PASSWORD`、`MYSQL_PASSWORD`、`MYSQL_ROOT_PASSWORD`、`RABBITMQ_PASSWORD`、`GRAFANA_ADMIN_PASSWORD` 是否仍以 `change_me`／`change-me`（不分大小寫）開頭；有任何一個是照抄範本的佔位值就中止部署（不會停服），訊息指出是哪個鍵。Grafana 不經 Go 程式讀取，只有這道檢查擋得住；RabbitMQ 密碼 World、Record 啟動時也會檢查（手動 `compose up` 沿用範本密碼同樣啟動失敗）
1b. 同樣在關服前，`preflight_migrate`（`scripts/lib/migrate.sh`）以與正式 migration 相同的旗標（`MIGRATION_PLAN_DIR`／`MIGRATION_ENVIRONMENT`／`MIGRATION_TIMEOUT`／`MIGRATION_LOCK_TIMEOUT`）跑 `datamigrate -mode validate`：不連資料庫、不帶 `MYSQL_DSN`，env 檔的 `MIGRATION_*` 寫錯或 `MIGRATION_PLAN_DIR` 不是已存在的宿主機絕對路徑，就在這裡中止（服務尚未停止），不會等到停服、備份、發布設定之後才失敗；也避免旗標錯誤時 datamigrate 印出含密碼 DSN 的預設值進 CI log
2. 停止六個服務（整批替換，不跨發版混跑），**停服後再備份一次 MySQL**（檔名帶 `-final` 後綴，例如 `mysql-20261020-120000-final.sql.gz`）：World 關機時會整批寫回玩家異動，關服前的備份不含這段，migration 損壞資料時要以停服後的最終狀態為還原點。這次備份落在維護窗口內，維護時間因此多出一次 `mysqldump` 的時間（資料量大時需留意 `deploy-production` 的 30 分鐘 job timeout）。
3. 發布 `config/data` 設定 artifact。World 啟動會比對 `current.json` 與映像內的 `release.json`，兩者必須同批更新，因此設定隨發版自動發布。
4. 執行 DB migration，把 `.env.prod` 的映像更新為 `images.env` 的 digest（沒有該檔時用 tag），啟動並等待六個服務 healthy（上限 `HEALTH_TIMEOUT` 秒，預設 180；必須是正整數秒，寫成 `3m` 這類時長字串會在任何停服動作之前以結束碼 2 中止，不會讓健康檢查與回滾被跳過而假成功）。維護結束後，若 `monitoring` profile 有在跑，會以 `--force-recreate --no-deps` 重建 `prometheus` 與 `alertmanager`：告警規則與 Alertmanager 設定是單檔 bind mount，換版後容器仍指向舊 inode，不重建就不會載入新規則。重建失敗只印 `WARN`（規則可能是舊的），不回滾已成功的應用部署。
5. 任一步失敗：映像（原樣還原部署前 `.env.prod` 的映像值）與設定 artifact 退回上一版，資料庫不還原。維護中（服務已停、尚未成功）收到 TERM／INT（job 逾時或手動取消）也同樣自動回滾，不會把服務停在關閉狀態；腳本因 INT 中斷以 130、因 TERM 中斷以 143 結束。回滾時設定 artifact 切回失敗不再被吞掉：印 `!!` 提示、回滾以非 0 結束，外層印 `ROLLBACK FAILED`（第十一輪；原本健康檢查通過仍印 rollback done）。缺 `IMAGE_TAG`、`REGISTRY` 等必填參數時印用法並以 2 結束。`deploy-production` 的 job timeout 仍是 30 分鐘。

自動發布只處理一般設定變更。拓樸或變數 migration（需 `-topology-migration`、`-variable-migration` 與離線搬遷）仍須依 [配置Artifact發布與回退](配置Artifact發布與回退.md) 手動處理，自動發布遇到會失敗並回退。

需要的設定：

- Runner：部署主機安裝 gitlab-runner（shell executor，標籤 `deploy`），具備 docker、git、bash、flock；Go 由 golang 容器提供，不需安裝。runner 設定**必須關閉「Run untagged jobs」並勾選「Lock to current projects」**：`.gitlab-ci.yml` 除了 `build-images`、`deploy-*`、`build-images-mr`、`backup-restore-drill` 之外的 job 都沒有 `tags`，deploy runner 若接未加標籤的 job，任何分支或 MR 的 `unit`、`lint` 等會以 shell executor 直接在 production 主機執行（`image:` 被忽略）；不需要 `build-images-mr` 時可再設成 Protected runner。
- CI/CD 變數（Protected）：`PROD_ENV_FILE`（`.env.prod` 的絕對路徑）；`REGISTRY`（未設定時使用 GitLab Container Registry）。`REGISTRY` 指向外部倉庫時還要設 `REGISTRY_USER`、`REGISTRY_PASSWORD`（Protected、Masked）：`scripts/ci-registry.sh` 依此登入／登出實際的 registry 主機，`build-images`、`deploy-staging`、`deploy-production` 共用；沒給帳密且主機不是 GitLab Registry 時略過登入，推送會因未授權失敗。
- GitLab 專案設定：**不要關閉**專案的「Use separate caches for protected branches」。protected 的 develop 與 unprotected 的 MR 分支快取分開，MR 第一次冷編譯（`.gocache` 的 `fallback_keys` 退回 `gocache-<job>-develop` 拿不到）是刻意接受的代價；關掉它，未受保護的 MR lint job（模組快取為 pull-push）就能寫入受保護分支與 tag pipeline 共用的快取。
- `trivy` job 與 `build-images` 的 `trivy image` 都快取 Trivy 漏洞資料庫（job cache／具名 volume `pja-trivy-cache`），避免每次發版下載 7 次。
- 部署主機以 `git submodule` 取得 `deps/PJA-Database`（job 已設 `GIT_SUBMODULE_STRATEGY: recursive`）。
