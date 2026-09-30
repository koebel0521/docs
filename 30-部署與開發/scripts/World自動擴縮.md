# `scripts/world-autoscale.ps1`

## 作用
根據 Prometheus 指標判斷 World 是否要擴縮。

## 目的
在爆量前先補容量，而不是等 queue 爆掉才處理。

## 流程
1. 讀 Prometheus 指標。
2. 算出目前 World 容量與 queue 壓力。
3. `-DryRun` 只出建議。
4. `-Apply` 才真的動作。

## 主要參數
- `-ComposeFile`
- `-WorldTemplate`
- `-PrometheusUrl`
- `-MinWorlds`
- `-MaxPlayersPerWorld`
- `-MaxTcpSendQueueDepth`
- `-MaxAppEventQueueDepth`
- `-Apply`
- `-DryRun`

## 相關文件
- [問題排查筆記](../../20-排障與監控/問題排查筆記.md)
