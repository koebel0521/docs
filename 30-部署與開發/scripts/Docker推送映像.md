# `scripts/docker-push-images.sh`

## 作用
把 `central / world / gate` 的 Docker image 推到 registry。

## 目的
把 build 好的 image 送到可部署的位置。

## 流程
1. 取 `REGISTRY` 與 `IMAGE_TAG`。
2. 依序 push 三個 image。

## 主要參數
- 透過環境變數控制：`REGISTRY`、`IMAGE_TAG`

## 相關文件
- [Docker發版](./Docker發版.md)
