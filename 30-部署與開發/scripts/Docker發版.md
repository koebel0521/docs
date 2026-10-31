# `scripts/docker-release.sh`

## 作用
串起 build / push / trace 的發版流程。

## 目的
把 Docker 發版做成單一入口，避免漏掉審批或 trace。

## 流程
1. 檢查 upgrade window lock。
2. 檢查 change approval stamp（須綁定目前 `HEAD`、未過期，發版成功後作廢；舊格式不接受）。
3. 工作樹有未提交修改（含未追蹤檔，`pkg/release/release.json` 除外）就拒絕發版：映像以工作樹（`COPY . .`）建置，但 `release.json` 與 trace 記的是 `HEAD`，內容會對不上 `server_commit`。
4. 以 `go run ./tools/pja releasemanifest generate`／`verify`（傳入完整 `HEAD` SHA，`verify` 加 `-config-repo deps/PJA-Config`，與 CI `release-info` 一致）產生並驗證 `pkg/release/release.json`（結束後還原開發版）；`config_version` 不是 `deps/PJA-Config` 子模組指標或其祖先（不可追溯）就失敗。
5. 跑 `docker-build-images.sh`。
6. 推送前對六個映像各跑一次 `trivy image --ignore-unfixed --exit-code 1 --severity HIGH,CRITICAL`（與 CI `build-images` 同一個釘 digest 的 trivy 映像，版本由 `TestToolchainVersionsAreConsistent` 比對）；有可修補的 HIGH／CRITICAL 漏洞就失敗，不推送。
7. 跑 `docker-push-images.sh`。
8. 寫 release trace 檔與 `.run/release/<tag>.release.json`。

## 主要參數
- 第一個位置參數：`image-tag`
- 第二個位置參數：`registry`，可省略（預設取環境變數 `REGISTRY`，再沒有則 `registry.example.com/project`）
- 環境變數 `ALLOW_OVERWRITE_TAG=1`：registry 已有同名 tag 時預設拒絕（會覆蓋已發出的映像），明確要覆蓋才設
- 環境變數 `UPGRADE_WINDOW_LOCK`（預設 `.run/upgrade-window.lock`）、`CHANGE_APPROVAL_FILE`（預設 `.run/change-approval/current.txt`）：覆寫升級窗口鎖與變更審批戳記的位置

## 相關文件
- [變更審批](./變更審批.md)
- [升級窗口](./升級窗口.md)
