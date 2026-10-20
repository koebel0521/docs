# `scripts/docker-release.sh`

## 作用
串起 build / push / trace 的發版流程。

## 目的
把 Docker 發版做成單一入口，避免漏掉審批或 trace。

## 流程
1. 檢查 upgrade window lock。
2. 檢查 change approval stamp。
3. 以 `go run ./tools/pja releasemanifest generate`／`verify`（傳入完整 `HEAD` SHA）產生並驗證 `pkg/release/release.json`（結束後還原開發版）。
4. 跑 `docker-build-images.sh` 與 `docker-push-images.sh`。
5. 寫 release trace 檔與 `.run/release/<tag>.release.json`。

## 主要參數
- 第一個位置參數：`image-tag`
- 第二個位置參數：`registry`，可省略（預設取環境變數 `REGISTRY`，再沒有則 `registry.example.com/project`）
- 環境變數 `UPGRADE_WINDOW_LOCK`（預設 `.run/upgrade-window.lock`）、`CHANGE_APPROVAL_FILE`（預設 `.run/change-approval/current.txt`）：覆寫升級窗口鎖與變更審批戳記的位置

## 相關文件
- [變更審批](./變更審批.md)
- [升級窗口](./升級窗口.md)
