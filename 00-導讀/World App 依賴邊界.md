# World App 依賴邊界

`cmd/world/internal/app` 是 World 的狀態核心，不直接持有 Redis client。

- `PlayerStore` 與 `GameStore` 維持由消費者使用的儲存介面。
- Redis 透過 `app.RedisStore` 最小角色介面注入。
- Redis、MySQL 與斷路器等實作由 `run.go` / `internal/store` 建構。
- 新增 App 行為時，先在 App 定義需要的最小方法集合，再由 adapter 實作。

這樣可以在不改變正式 Redis 實作的情況下，以 fake 取代外部儲存測試 World 流程。
