# SLO 值班流程

這份 runbook 對應 `指標面板` 的 SLO summary 與 Prometheus alert。
目標是讓值班時先看對的圖，再決定要不要往下鑽 handler、TCP、或狀態對帳。

## 快速決策

1. 先開 [本機監控](./本機監控.md)
2. 先看 `tools/private/dephealth`（私人工具，見下方「沒有私人工具時」）
3. 再看 `tools/private/opsdiag`（同上）
4. 若看到 `mismatch` / `error`，先進 [狀態對帳流程](./狀態對帳流程.md)
5. 若看到 `status=degraded`，先看 `Capacity Budget`
6. 最後才進 SLO 類別與 handler-level log

## 沒有私人工具時

本 runbook 與[問題排查筆記](./問題排查筆記.md)提到的 `tools/private/*`（dephealth、opsdiag、protocoldiag、supportdiag、playersnapshot、consistencysnapshot、staterepair、datasurgeon 等）以及 `go run ./tools/private/pja ...` 都是**私人工具**：放在被 `.gitignore` 忽略的 `tools/private/`，不在版控，換一台機器或乾淨 clone 沒有。版控內的 `tools/pja` 只有 `configartifact`、`configaudit`、`releasemanifest` 三個子命令。沒有私人工具時，直接打各服務的診斷埠（Central 6060、Gate 6061、World 6062、Lobby 6064、Record 6065、Chat 6066；正式環境的診斷埠只綁容器內，`compose.prod.yml` 沒有發布 606x 到主機，所以下表的 `curl -s localhost:<埠>/...` 在正式環境要改成 `docker exec <容器> wget -qO- http://127.0.0.1:<埠>/...`（映像只有 `wget`），或經同網段的 Prometheus 查；本機 dev 堆疊才能直接 curl）：

| 私人工具 | 等價的版控內做法 |
|------|------|
| `dephealth`（readiness、queue 深度） | `curl -s localhost:<埠>/ready`；`curl -s localhost:<埠>/metrics \| grep -E 'game_event_queue_depth\|game_msg_queue_dropped_total'`；或 Grafana／Prometheus 的對應面板 |
| `opsdiag`（readiness、路由查核、關鍵 metrics） | 各服務 `/ready`；Gate `curl -s -u "$DEBUG_PPROF_BASIC_AUTH" "localhost:6061/diag/session?playerID=<id>"`、`/diag/player?playerID=<id>`（Gate `/diag/session`、`/diag/player` 與 pprof 一樣走 basic auth，帳密為 `.env.prod` 的 `DEBUG_PPROF_BASIC_AUTH`，值為 `user:pass`）；PromQL 看 SLO 面板 |
| `protocoldiag` | `curl -s localhost:<埠>/diag/protocol`（Central、Gate、World；不需 basic auth） |
| `supportdiag`／`playersnapshot` | Gate `/diag/player`、`/diag/session`（同樣要 `-u`）；Redis `GET player:{playerID}`（登入狀態快照 JSON） |
| `staterepair`／`datasurgeon` | 沒有等價的版控內工具；修 Redis 衍生資料前先備份（[[Docker備份還原]]），依 [Redis 鍵值結構](../90-參考附件/Redis鍵值結構.md) 手動處理，或請持有私人工具的人執行 |

## 先決定看哪個 SLO

各 SLO 類別的詳細判讀、面板路徑與常見根因，統一寫在 [指標面板.md](./指標面板.md) 的對應章節，這裡只列出入口對照：

| 症狀 | 指標面板 章節 | 必要時搭配 |
|------|----------------------|-----------|
| Dependency Health 黃了 | 依賴健康 | [資源預算](./資源預算.md)、opsdiag |
| Login Availability 掉了 | Login Outcomes / Login Failure Ratio | 問題排查筆記、連線與登入流程 |
| Logout Availability 掉了 | Logout Outcomes | 狀態對帳流程 |
| Recovery Correctness 掉了 | Recovery Validation / State Audit | [狀態對帳流程](./狀態對帳流程.md) |
| Reconnect Success 掉了 | Reconnect Outcomes / TCP queue | 本機監控 |
| Capacity Budget 快爆了 | Capacity Budget | [資源預算](./資源預算.md)、opsdiag |

先開 [指標面板](./指標面板.md) 找到對應 SLO 類別，再依該章節的判讀順序處理。不再在這裡重複。

## SLO 值班順序

1. 看 `SLO Summary`
2. 找掉線的 SLO 類別，對照上方表格進 [指標面板](./指標面板.md) 對應章節
3. 若 `Dependency Health` 已 degraded，先看 [資源預算](./資源預算.md) 與 `opsdiag`
4. 若是狀態不一致，再看 [狀態對帳流程](./狀態對帳流程.md)
5. 最後才進 handler-level log 與 pprof

## 一句話

- `ready` 壞 -> 看服務
- `degraded` -> 看 budget
- `mismatch / error` -> 看 audit

## 建議原則

- SLO 告警是入口，不是結論
- 先看趨勢，再看單點 log
- 先看 `summary`，再看原始數字，最後才手動重算

## 告警索引

本文件是下列告警的 `runbook_url`。各告警的完整條件、`for`、等級與判讀寫在 [指標面板](./指標面板.md)（以告警名稱搜尋）；處置順序依本文件。

| 告警 | 等級 | 說明 |
| --- | --- | --- |
| `GameLoginAvailabilityLow` | warning | 登入可用率低於 99%，先看 login outcomes 與 handler latency；`client_closed`（玩家自己斷線）不算失敗也不進分母，`session_expired` 只代表 pending 逾時 |
| `GameLoginCentralBusy` | warning | Central 登入併發滿載而拒絕登入，先看開服／大批重連尖峰與 bcrypt 併發 |
| `GameLoginGateUnavailable` | warning | Gate 持續回 service_unavailable（`reason` 為 `ban_check_failed` 或 `no_central`），ban_check_failed 先看 Redis，no_central 先看 Central 連線 |
| `GameLogoutAvailabilityLow` | warning | 登出可用率低於 99%，先看 logout outcomes（登出 pending 期間的重送已改計 `game_request_ignored_total{flow="logout",cause="duplicate_pending"}`，不在此分母內；Gate 本地限流擋下的登出 `cause="rate_limited"` 同樣改計 `game_request_ignored_total`，不在 `game_logout_total` 分母） |
| `TCPReconnectSuccessLow` | warning | {{ $labels.instance }} 連 {{ $labels.service }} 的 S2S 重連成功率低於 99%，先看 reconnect outcomes 與 socket queue overflow |
| `GameLoginFailureRatioHigh` | warning | 5 分鐘內登入失敗率超過 10% |
| `TCPReconnectFailuresSustained` | warning | {{ $labels.instance }} 連 {{ $labels.service }} 重連持續失敗且沒有成功，依賴服務可能斷線或網路異常 |
| `GameHandlerP95High` | warning | handler {{ $labels.handler }} 的 P95 超過 0.2 秒 |
| `GameHandlerP99Critical` | page | handler {{ $labels.handler }} 的 P99 超過 0.5 秒，明顯影響玩家體驗 |
| `GameHTTPErrorRatioHigh` | warning | {{ $labels.job }} 路由 {{ $labels.route }} 的 HTTP 5xx 比例超過 1% |
| `GameHTTPLatencyP99High` | warning | {{ $labels.job }} 的 HTTP 請求 P99 超過 1 秒 |
