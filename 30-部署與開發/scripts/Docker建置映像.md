# `scripts/docker-build-images.sh`

## 作用
建置 `central / world / gate / chat / lobby / record` 六個服務的 Docker image。

## 目的
發版前先確認六個自建 image 都能成功 build。

## 流程
1. 取 `REGISTRY`、`IMAGE_TAG` 與 `APT_REFRESH`。
2. 依序 build `central`、`world`、`gate`、`chat`、`lobby`、`record`（各自帶除錯埠作為映像的 `HEALTHCHECK_PORT`）。
3. 印出已建好的 image 列表。

## 主要參數
- 透過環境變數控制：`REGISTRY`、`IMAGE_TAG`、`APT_REFRESH`（預設取 `CI_PIPELINE_ID`，否則當天日期；讓 runtime 階段的 apt upgrade 層每條 pipeline 或每天失效一次）

正式 `compose.prod.yml` 的 `CHAT_IMAGE`、`RECORD_IMAGE` 為必填，所以六個映像都要建置並推送。

## 相關文件
- [Docker發版](./Docker發版.md)
