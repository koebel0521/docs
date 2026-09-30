# `scripts/docker-deps-up.sh`

## 作用
啟動 Docker 依賴堆疊。

## 目的
先把 MySQL、Redis、RabbitMQ、Prometheus、Grafana 這類共用依賴起來。

## 流程
1. 跑 `docker compose -f compose.deps.yml up -d`。
2. 印出常用連線位址。

## 主要參數
- 無。

## 相關文件
- [Docker全堆疊啟動](./Docker全堆疊啟動.md)
