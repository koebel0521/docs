# World App 依賴邊界

`cmd/world/internal/app` 是 World 的狀態核心，不直接持有 Redis client。

- App 只認窄介面：玩家狀態用 `playerstore.Store`；常駐玩家遊戲資料只透過 `playerDataStore`（卸載、離線卸載掃描、關機寫回與落地四類方法，`users_types.go`）使用，其餘玩家資料操作由組合根直接交給用到它的元件，App 不經手。已刪（2026-10-31）的舊名 `gamestore.Store` 大介面與舊名 `Coordinator.gameStore`；`App.PlayerData()` 只供組合根確認 App 與 `services.Options.GameStore` 是同一個實例（`services.GameStore` 由消費端組合小介面，唯一正式實作是 `*gamestore.Memory`）。
- App 不使用 Redis，也不持有或轉手 Redis client（`internal/app` 不再 import `pkg/redis`，由守門測試把關）：GM 發放佇列由 `services` 以 `Options.Redis`（原 `DataConfigOptions`）取得。
- request journal（2026-10-30 起只剩驗證、擲骰與建築升級三種，登入、登出 journal 已刪）的持久後援以 `cmd/world/internal/store/requestjournal` 的 `KV` 介面注入：組合根以 `requestjournal.NewRedisKV(client)` 包裝後傳給 `app.WithPersistentRequestJournals`（Redis 為必要依賴：缺少時啟動失敗或 panic，不再有傳 nil、只用本機快取的退路）；App 只認 `KV`，不知道背後是 Redis。
- 玩家卸載後的清理以回呼注入：`WithRequiredPlayerForgetter()` 是無參數的旗標選項（宣告正式組合根必須綁定，`cmd/world/run.go` 建 App 時帶入）；回呼本身由 App 方法 `BindPlayerForgetter(forget)` 延遲綁定（變數服務晚於 App 才建好），由 `services.wireWorldCallbacks` 呼叫並傳入 `variables.ForgetPlayer`；旗標已宣告而未綁定就在第一次卸載玩家時 panic（不靜默略過、不在建構時失敗），由 `services_wire_callbacks_test.go` 守門（拿掉接線即失敗）；未宣告旗標（單元測試）時 nil 回呼略過。回呼指向 `variables.ForgetPlayer`（刪除變數服務的遷移記錄，見[玩家變數](./玩家變數.md)）；`EvictPlayerData` 與離線卸載成功後呼叫它，App 不直接依賴變數服務。
- World 啟動已強制要求 MySQL 與記憶體版 `GameStore`，舊戰鬥實作已整批刪除，新戰鬥依 PJA-Docs/戰鬥規則.md 重寫；登入流程（`login.New(app, services, activity)`）只要求一個 `loginActivityToucher`（`TouchLoginActivity`，由組合根注入 Central 事件處理器持有的 store），不依賴 gamestore 大介面。背景工作與 GM 發道具直接收 Start 已驗證的 `GameStore`，變數服務由 Start 區域變數直接傳給盤面處理器、設定熱重載與 Gate 事件處理器，App 不再保留轉手欄位。盤面 router 與 App 的 nil 防護只為了讓單元測試以簡化 router 測單一指令，正式環境不會走到。
- 事件記錄輸出端同樣是窄介面：App 的 `recordSink` 只有 `Publish`、`PublishSync`、`IsConnected` 三個方法，`App` 不 import `pkg/recordmq`（連帶不編入 AMQP 用戶端）；正式實作 `*recordmq.AsyncPublisher` 由 `services.Run` 經 `SetRecordSink` 接上，事件封包的建構已搬到 `cmd/world/internal/outbox/envelope.go`（2026-10-31）；「sink 尚未建立而丟棄」的計數由 `SetRecordNotReadyCounter` 注入回呼（`services.Run` 傳入 `recordmq.CountNotReadyDrop`），守門 `TestWorldAppDoesNotDependOnRecordMQ`。
- Redis、MySQL 與斷路器等實作由 `run.go` / `cmd/world/internal/store` 建構。
- 新增 App 行為時，先在 App 定義需要的最小方法集合，再由 adapter 實作。

這樣可以在不改變正式 Redis 實作的情況下，以 fake 取代外部儲存測試 World 流程。
