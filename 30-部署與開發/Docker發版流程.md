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

## 3. 建議 release 步驟

### 手動步驟

1. 決定版本號；`go run ./tools/pja releasemanifest generate -release <tag> -commit <SHA>` 依實際產物寫出 `release.json`
2. `go run ./tools/pja releasemanifest verify` 驗證 server/config/schema/protocol 對應；需先 `git submodule update --init`（migration 目錄不存在時 `generate`／`verify` 都會報錯，不會當成沒有 schema）
3. build image
4. push image
5. 完成 MySQL backup / snapshot
6. 執行一次性的 `deps/PJA-Database/scripts/migrate-prod.sh`
7. 確認 migration 成功後更新 deploy 主機上的 `.env.prod`
8. 執行 production deploy：同一次發版的所有服務整批替換，不可新舊發版混跑（服務之間的 `ServiceHello` 比對發版身分，不一致會斷線，見 [版本握手與強制更新](../10-核心流程/版本握手與強制更新.md#服務之間)）
9. 跑 production smoke test
10. 若失敗則只 rollback server image，保留已向前的 database schema

### 使用 helper script

可直接執行：

```bash
./scripts/docker-release.sh 1.0.0 registry.example.com/project
```

或直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerrelease -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
```

若只是先產審批 stamp，也可以先跑：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode changeapproval -ChangeApprovalChangeType release -ChangeApprovalTarget 1.0.0 -ChangeApprovalOwner alice
```

這個腳本會：

1. 設定 `IMAGE_TAG`
2. 檢查升級窗口鎖與變更審批戳記，再以 `go run ./tools/pja releasemanifest generate`／`verify` 依實際產物寫出並驗證 `pkg/release/release.json`（嵌入映像；腳本結束後還原該檔）
3. 執行 [docker-build-images.sh](../../scripts/docker-build-images.sh)
4. 執行 [docker-push-images.sh](../../scripts/docker-push-images.sh)

若只想分開跑，也可以直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerbuild -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerpush -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.example.com/project
```

## 4. deploy 主機更新內容

release 完成後，deploy 主機至少要更新：

- `.env.prod` 內的 `IMAGE_TAG`
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

```bash
docker compose --env-file .env.prod -f compose.prod.yml up -d
```

若有 monitoring：

```bash
docker compose --env-file .env.prod -f compose.prod.yml --profile monitoring up -d
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

1. 將 `.env.prod` 的 image tag 改回上一版
2. 重新執行 deploy
3. 不要對 production 執行 migration rollback
4. 若資料也出問題，依 [[Docker備份還原]] 從已驗證的 backup/snapshot restore，並先隔離流量

rollback 範例：

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
docker compose --env-file .env.prod -f compose.prod.yml up -d
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

推 `vX.Y.Z` 標籤（請設為 Protected tag）後，`.gitlab-ci.yml` 依序執行：

1. 既有檢查全部通過，`release-info` 產出 `release.json`。
2. `build-images`：建置並推送六個映像（tag 為去掉 `v` 的版本號）。
3. `deploy-staging`：預留。目前無第二台主機，設定 `STAGING_ENABLED=true` 與 `STAGING_ENV_FILE` 才啟用。
4. `deploy-production`：手動按下後執行 [ci-deploy.sh](../../scripts/ci-deploy.sh)。

`ci-deploy.sh` 預期關服維護，流程固定：

1. 先拉新映像，備份 MySQL。
2. 停止六個服務（整批替換，不跨發版混跑）。
3. 發布 `config/data` 設定 artifact。World 啟動會比對 `current.json` 與映像內的 `release.json`，兩者必須同批更新，因此設定隨發版自動發布。
4. 執行 DB migration，更新 `.env.prod` 的映像 tag，啟動並等待六個服務 healthy。
5. 任一步失敗：映像與設定 artifact 退回上一版，資料庫不還原。

自動發布只處理一般設定變更。拓樸或變數 migration（需 `-topology-migration`、`-variable-migration` 與離線搬遷）仍須依 [配置Artifact發布與回退](配置Artifact發布與回退.md) 手動處理，自動發布遇到會失敗並回退。

需要的設定：

- Runner：部署主機安裝 gitlab-runner（shell executor，標籤 `deploy`），具備 docker、git、bash、flock；Go 由 golang 容器提供，不需安裝。
- CI/CD 變數（Protected）：`PROD_ENV_FILE`（`.env.prod` 的絕對路徑）；`REGISTRY`（未設定時使用 GitLab Container Registry）。
- 部署主機以 `git submodule` 取得 `deps/PJA-Database`（job 已設 `GIT_SUBMODULE_STRATEGY: recursive`）。
