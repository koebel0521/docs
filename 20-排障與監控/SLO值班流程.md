# SLO 值班流程

這份 runbook 對應 `指標面板` 的 SLO summary 與 Prometheus alert。
目標是讓值班時先看對的圖，再決定要不要往下鑽 handler、TCP、或狀態對帳。

## 快速決策

1. 先開 [本機監控](./本機監控.md)
2. 先看 `tools/dephealth`
3. 再看 `tools/opsdiag`
4. 若看到 `mismatch` / `error`，先進 [狀態對帳流程](./狀態對帳流程.md)
5. 若看到 `status=degraded`，先看 `Capacity Budget`
6. 最後才進 SLO 類別與 handler-level log

## 先決定看哪個 SLO

各 SLO 類別的詳細判讀、面板路徑與常見根因，統一寫在 [指標面板.md](./指標面板.md) 的對應章節，這裡只列出入口對照：

| 症狀 | 指標面板 章節 | 必要時搭配 |
|------|----------------------|-----------|
| Dependency Health 黃了 | 依賴健康 | [資源預算](./資源預算.md)、opsdiag |
| Login Availability 掉了 | Login Outcomes / Login Failure Ratio | 問題排查筆記、連線與登入流程 |
| Logout Availability 掉了 | Logout Outcomes / Replay Cache | 狀態對帳流程 |
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
