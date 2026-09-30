# `scripts/docker-build-images.sh`

## 作用
建置 `central / world / gate` 的 Docker image。

## 目的
發版前先確認三個主要 image 都能成功 build。

## 流程
1. 取 `REGISTRY` 與 `IMAGE_TAG`。
2. 依序 build `central`、`world`、`gate`。
3. 印出已建好的 image 列表。

## 主要參數
- 透過環境變數控制：`REGISTRY`、`IMAGE_TAG`

## 相關文件
- [Docker發版](./Docker發版.md)
