# `scripts/world-autoscale.ps1`

## 作用
根據 Prometheus 指標判斷 World 是否要擴縮。

## 目的
在爆量前先補容量，而不是等 queue 爆掉才處理。

## 流程
1. 讀 Prometheus 指標。
2. 任一指標查不到就中止，不會當成 0 而縮容。
3. 算出目前 World 容量與 queue 壓力。
4. `-DryRun`（或沒有 `-Apply`）只出建議，但同樣以 `docker ps` 查實際 World 容器（唯讀；docker 因此是必要條件），計畫與 `-Apply` 一致；縮容前的抓取數比對在 dry-run 也執行，不一致只印警告不中止（第十一輪；原本 dry-run 恆假設 0 個 World、只會印擴容）。
5. `-Apply` 才真的動作；縮容會停掉 World 容器、中斷其玩家，必須另加 `-AllowScaleDown`，並依 World ID 數字由大到小停。縮容前比對 `count(up{job="world"}==1)` 與實際 World 容器數，不一致（有 World 沒被抓到指標）就中止，不會依偏低的在線數縮容。
6. 只有 `-Apply`（且非 `-DryRun`）時，才把擴出的 `project-full-world-N:6062` 寫入 `docker/monitoring/targets/` 目錄下的 world-autoscale.json（執行期產生，不進版控）（先寫暫存檔再改名），Prometheus 經 file_sd 自動抓取新 World。

## 連線預算檢查
算出 desired World 數後，若超過 `floor((MysqlMaxConnections - ConnsOtherServices - ConnsReserve) / ConnsPerWorld)`（預設 300 時為 3 個 World）就直接拋錯拒絕，`-DryRun` 也一樣；World 數超過預算會讓各池補建連線撞 MySQL Error 1040，斷路器打開後載入與寫回整批失敗。要擴更多 World 先調高 `compose.full.yml`／`compose.prod.yml` 的 `--max-connections`，並同步 `-MysqlMaxConnections`。

## 主要參數
- `-ComposeFile`
- `-WorldTemplate`
- `-PrometheusUrl`
- `-MinWorlds`
- `-MaxPlayersPerWorld`
- `-MaxTcpSendQueueDepth`
- `-MaxAppEventQueueDepth`
- `-MysqlMaxConnections`（預設 300，須與 compose 的 `--max-connections` 一致）、`-ConnsPerWorld`（預設 56：主池 40 加載入池 16）、`-ConnsOtherServices`（預設 75：Central、Chat、Record 各 25）、`-ConnsReserve`（預設 10）：連線預算，見下方「連線預算檢查」
- `-Apply`
- `-AllowScaleDown`
- `-DryRun`

## 相關文件
- [問題排查筆記](../../20-排障與監控/問題排查筆記.md)
