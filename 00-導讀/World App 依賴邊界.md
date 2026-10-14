# World App 依賴邊界

`cmd/world/internal/app` 是 World 的狀態核心，不直接持有 Redis client。

- `PlayerStore` 與 `GameStore` 維持由消費者使用的儲存介面（程式中的型別名為 `playerstore.Store`、`gamestore.Store`，文件沿用概念名稱）。
- App 不使用 Redis，也不持有或轉手 Redis client：戰鬥候選索引在 `cmd/world/run.go` 的 `newCombatStores` 建立，GM 發放佇列由 `services` 以 `DataConfigOptions.Redis` 取得。
- Redis、MySQL 與斷路器等實作由 `run.go` / `internal/store` 建構。
- 新增 App 行為時，先在 App 定義需要的最小方法集合，再由 adapter 實作。

這樣可以在不改變正式 Redis 實作的情況下，以 fake 取代外部儲存測試 World 流程。
