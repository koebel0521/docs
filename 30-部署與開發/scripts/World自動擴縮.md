# `scripts/world-autoscale.ps1`

## 作用
根據 Prometheus 指標判斷 World 是否要擴縮。

## 目的
在爆量前先補容量，而不是等 queue 爆掉才處理。

## 流程
1. 讀 Prometheus 指標。
2. 任一指標查不到就中止，不會當成 0 而縮容。
3. 算出目前 World 容量與 queue 壓力。
4. `-DryRun` 只出建議。
5. `-Apply` 才真的動作；縮容會停掉 World 容器、中斷其玩家，必須另加 `-AllowScaleDown`，並依 World ID 數字由大到小停。縮容前比對 `count(up{job="world"}==1)` 與實際 World 容器數，不一致（有 World 沒被抓到指標）就中止，不會依偏低的在線數縮容。
6. 只有 `-Apply`（且非 `-DryRun`）時，才把擴出的 `project-world-N:6062` 寫入 `docker/monitoring/targets/world-autoscale.json`（先寫暫存檔再改名），Prometheus 經 file_sd 自動抓取新 World。

## 主要參數
- `-ComposeFile`
- `-WorldTemplate`
- `-PrometheusUrl`
- `-MinWorlds`
- `-MaxPlayersPerWorld`
- `-MaxTcpSendQueueDepth`
- `-MaxAppEventQueueDepth`
- `-Apply`
- `-AllowScaleDown`
- `-DryRun`

## 相關文件
- [問題排查筆記](../../20-排障與監控/問題排查筆記.md)
