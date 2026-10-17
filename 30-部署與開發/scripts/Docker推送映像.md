# `scripts/docker-push-images.sh`

## 作用
把六個服務（central、world、gate、chat、lobby、record）的 Docker image 推到 registry，並記錄各 image 的 digest。

## 目的
把 build 好的 image 送到可部署的位置。

## 流程
1. 取 `REGISTRY` 與 `IMAGE_TAG`。
2. 依序 push 六個 image。
3. 把 `IMAGE_TAG` 與各 image 的 `registry/project-<app>@sha256:...` 寫入 `IMAGES_FILE`（預設 `images.env`）；CI 存成 artifact，`ci-deploy.sh` 以 digest 部署與回滾。

## 主要參數
- 透過環境變數控制：`REGISTRY`、`IMAGE_TAG`、`IMAGES_FILE`

## 相關文件
- [Docker發版](./Docker發版.md)
