# `scripts/docker-release.sh`

## 作用
串起 build / push / trace 的發版流程。

## 目的
把 Docker 發版做成單一入口，避免漏掉審批或 trace。

## 流程
1. 檢查 upgrade window lock。
2. 檢查 change approval stamp。
3. 跑 `docker-build-images.sh` 與 `docker-push-images.sh`。
4. 寫 release trace 檔。

## 主要參數
- 第一個位置參數：`image-tag`
- 第二個位置參數：`registry`，可省略

## 相關文件
- [變更審批](./變更審批.md)
- [升級窗口](./升級窗口.md)
