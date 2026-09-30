# `scripts/docker-full-up.sh`

## 作用
啟動完整 Docker 堆疊。

## 目的
一次拉起所有本機服務，方便整體驗證。

## 流程
1. 檢查 Docker 是否存在。
2. 跑 `docker compose -f compose.full.yml up -d --build`。
3. 印出 gate、record、RabbitMQ、Grafana、Prometheus 的入口。

## 主要參數
- 無。

## 相關文件
- [生產冒煙測試](./生產冒煙測試.md)
