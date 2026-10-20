# `scripts/docker-deps-up.sh`

## 作用
啟動 Docker 依賴堆疊。

## 目的
先把 MySQL、Redis、RabbitMQ、Prometheus、Grafana 這類共用依賴起來。

## 流程
1. 以腳本位置推出倉庫根目錄（`ROOT_DIR`），跑 `docker compose -f "$ROOT_DIR/docker/compose/compose.deps.yml" up -d`；不必在倉庫根目錄執行（第十一輪）。
2. 印出常用連線位址。

## 主要參數
- 無。

## 相關文件
- [Docker全堆疊啟動](./Docker全堆疊啟動.md)
