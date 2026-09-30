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

1. 同一次 release，`central / world / gate` 使用同一個 tag
2. production 不直接部署 `latest`
3. rollback 時回到上一個已知穩定 tag

## 2. release 產物

每次 release 至少應有：

- `project-central:<tag>`
- `project-world:<tag>`
- `project-gate:<tag>`
- 對應 commit id
- deploy 時使用的 `.env.prod` 版本紀錄
- release manifest（server commit/image、database schema/migrations、Luban config、protocol、fixtures）
- migration plan checksum 與 schema migration 狀態

## 3. 建議 release 步驟

### 手動步驟

1. 決定版本號並完成 `release/<tag>.yaml`
2. 以 manifest 驗證 server/config/schema/protocol 對應
3. build image
4. push image
5. 完成 MySQL backup / snapshot
6. 執行一次性的 `../PJA-Database/scripts/migrate-prod.sh`
7. 確認 migration 成功後更新 deploy 主機上的 `.env.prod`
8. 執行 production deploy
9. 跑 production smoke test
10. 若失敗則只 rollback server image，保留已向前的 database schema

### 使用 helper script

可直接執行：

```bash
./scripts/docker-release.sh 1.0.0 registry.github.com/koebel0521/project
```

或直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode dockerrelease -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.github.com/koebel0521/project
```

若只是先產審批 stamp，也可以先跑：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode changeapproval -ChangeApprovalChangeType release -ChangeApprovalTarget 1.0.0 -ChangeApprovalOwner alice
```

這個腳本會：

1. 設定 `IMAGE_TAG`
2. 驗證 `release/<tag>.yaml`
3. 執行 [docker-build-images.sh](../../scripts/docker-build-images.sh)
4. 執行 [docker-push-images.sh](../../scripts/docker-push-images.sh)

若只想分開跑，也可以直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode dockerbuild -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.github.com/koebel0521/project
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode dockerpush -DockerReleaseImageTag 1.0.0 -DockerReleaseRegistry registry.github.com/koebel0521/project
```

## 4. deploy 主機更新內容

release 完成後，deploy 主機至少要更新：

- `.env.prod` 內的 `IMAGE_TAG`
- `.env.prod` 內的：
  - `CENTRAL_IMAGE`
  - `WORLD_IMAGE`
  - `GATE_IMAGE`

例如：

```text
IMAGE_TAG=1.0.0
CENTRAL_IMAGE=registry.github.com/koebel0521/project/project-central:1.0.0
WORLD_IMAGE=registry.github.com/koebel0521/project/project-world:1.0.0
GATE_IMAGE=registry.github.com/koebel0521/project/project-gate:1.0.0
```

同時建議把下列 trace 一起留存：

- build hash
- commit hash
- config hash
- manifest path/hash
- release 時間

`scripts/docker-release.sh` 會把這些資訊寫到 `.run/release/<tag>.txt`。

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
CENTRAL_IMAGE=registry.github.com/koebel0521/project/project-central:0.9.9
WORLD_IMAGE=registry.github.com/koebel0521/project/project-world:0.9.9
GATE_IMAGE=registry.github.com/koebel0521/project/project-gate:0.9.9
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

## 9. GoReleaser 平行發版（GitHub Release）

除了 Docker 映像檔發版，專案另有 GoReleaser 發版流程：

- **觸發條件**：推送 `v*` tag 至 GitHub
- **產物**：GitHub Release + 各平台 binary archive（tar.gz）+ checksum
- **不含 Docker**：映像檔仍由 `scripts/docker-release.sh` 管理

`.goreleaser.yml` 定義了 7 個服務（central / gate / world / user / lobby / chat / record）的 cross-compile 設定。

### 使用方式

```bash
git tag v1.0.0
git push origin v1.0.0
# GitHub Actions 自動建立 Release + binary artifacts
```

之後可從 GitHub Releases 頁面下載 binary，或繼續用 Docker 發版流程。

## 10. 不建議的做法

- 直接部署 `latest`
- 不記錄 commit 與 image tag 對應
- deploy 後不跑 smoke test
- rollback 只憑印象，不憑版本
