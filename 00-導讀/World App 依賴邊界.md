# World App 依賴邊界

`cmd/world/internal/app` 是 World 的狀態核心，不直接持有 Redis client。

- `PlayerStore` 與 `GameStore` 維持由消費者使用的儲存介面（程式中的型別名為 `playerstore.Store`、`gamestore.Store`，文件沿用概念名稱）。
- App 不使用 Redis，也不持有或轉手 Redis client（`internal/app` 不再 import `pkg/redis`，由守門測試把關）：戰鬥候選索引在 `cmd/world/run.go` 的 `newCombatStores` 建立，GM 發放佇列由 `services` 以 `Options.Redis`（原 `DataConfigOptions`）取得。
- request journal（2026-10-30 起只剩驗證、擲骰與建築升級三種，登入、登出 journal 已刪）的持久後援以 `cmd/world/internal/store/requestjournal` 的 `KV` 介面注入：組合根以 `requestjournal.NewRedisKV(client)` 包裝後傳給 `app.WithPersistentRequestJournals`（Redis 為必要依賴：缺少時啟動失敗或 panic，不再有傳 nil、只用本機快取的退路）；App 只認 `KV`，不知道背後是 Redis。
- 玩家卸載後的清理以回呼注入：`WithRequiredPlayerForgetter()` 是無參數的旗標選項（宣告正式組合根必須綁定，`cmd/world/run.go` 建 App 時帶入）；回呼本身由 App 方法 `BindPlayerForgetter(forget)` 延遲綁定（變數服務晚於 App 才建好），由 `services.wireWorldCallbacks` 呼叫並傳入 `variables.ForgetPlayer`；旗標已宣告而未綁定就在第一次卸載玩家時 panic（不靜默略過、不在建構時失敗），由 `services_wire_callbacks_test.go` 守門（拿掉接線即失敗）；未宣告旗標（單元測試）時 nil 回呼略過。回呼指向 `variables.ForgetPlayer`（刪除變數服務的遷移記錄，見[玩家變數](./玩家變數.md)）；`EvictPlayerData` 與離線卸載成功後呼叫它，App 不直接依賴變數服務。
- World 啟動已強制要求 MySQL 與 `GameStore`，所以登入流程（`Coordinator.gameStore` 必填）與戰鬥（`newCombatStores`、`startCombat`、`newCombatMatchService`）**沒有儲存層就啟動失敗**，不再有「沒有 GameStore 時在事件迴圈同步完成登入」或「沒有戰鬥儲存層就靜默停用 PvP」的分支；背景工作與 GM 發道具直接收 Start 已驗證的 `GameStore`，變數服務由 Start 區域變數直接傳給盤面處理器、設定熱重載與 Gate 事件處理器，App 不再保留轉手欄位。盤面 router 與 App 的 nil 防護只為了讓單元測試以簡化 router 測單一指令，正式環境不會走到。
- Redis、MySQL 與斷路器等實作由 `run.go` / `internal/store` 建構。
- 新增 App 行為時，先在 App 定義需要的最小方法集合，再由 adapter 實作。

這樣可以在不改變正式 Redis 實作的情況下，以 fake 取代外部儲存測試 World 流程。
