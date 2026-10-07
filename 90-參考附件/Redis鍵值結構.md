# Redis Key Schema 總表

本文件列出專案所有 Redis key prefix 與其 owner。TTL 以程式常數為準（括號內為常數名稱）；Redis TTL 一律以 PEXPIRE／SET PX 毫秒精度設定。

## `lobby:*` — 服務探索（lobbyreporter）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `lobby:gate:{gateID}` | Hash | Gate | 25s（`keyTTL`，每次回報刷新） | Gate 地址與負載 |
| `lobby:central:{centralID}` | Hash | Central | 25s（`keyTTL`） | Central 地址 |
| `lobby:world:{worldID}` | Hash | World | 25s（`keyTTL`） | World 地址與容量 |
| `lobby:gates`、`lobby:centrals`、`lobby:worlds` | Set | 各服務 | 無 | 各類服務的 ID 索引（`IndexKey`），成員為服務 ID；服務停止時移除 |

## `player:*` / `world:*` — 玩家狀態（playerstoreredis）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `player:{playerID}` | String (JSON) | World | 24 小時（`playerTTL`，每次寫入刷新） | 玩家登入狀態快照（gateID, worldID） |
| `world:{worldID}:players` | Set | World | 無 | World 上所有玩家的索引（供 restore）；刪除或改存到其他 World 時由 Lua 腳本移除成員，restore 時清掉過期成員 |

## `gm:*` — GM 操作

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `gm:ban:{account}` | String | Lobby (admin) | 封禁期間（`duration` 參數，預設 30 天 `defaultBanTTL`） | 帳號封禁標記（key 存在 = 被封禁） |
| `gm:kick:{gateID}` | List | Lobby (admin) → Gate | 消費後移除 | GM 踢人佇列，Gate 用 BLPOP 消費 |
| `gm:grant_item:{worldID}` | List | Lobby (admin) → World | 消費後移除 | GM 發道具佇列，World 用 BRPopLPUSH 消費 |
| `gm:grant_item:processing:{worldID}` | List | World | 消費期間 | 處理中指令（BRPOPLPUSH 從佇列移入，處理完移除，避免重複消費） |
| `gm:grant_item:retry:{worldID}` | ZSet | World | 重試期間 | 失敗指令重試佇列，按時間排程 |
| `gm:grant_item:capability:{worldID}` | String | World | 永久（on/off） | World 是否支援 GM 發道具 |
| `gm:grant_item:deadletter:{worldID}` | List | World | 7 天（`GrantItemDeadLetterTTL`） | 超過重試次數的死信佇列 |

佇列中的指令是 `grantitem.Command` 的 JSON，World 以嚴格模式解碼（拒絕未知欄位與尾部資料），無法解碼的指令原文放進死信保留 7 天。`Command` 新增欄位時必須先升級所有 World，再升級寫入佇列的 Lobby；反過來升級時，舊 World 會把新格式的指令全部放進死信，升級 World 與工具後再以 `tools/deadletterreplay` 重送（舊版工具同樣無法解碼，會直接報錯）；`tools/datasurgeon -mode deadletter` 可列出或清除無法解碼的項目。

## `lobby:notice:*` — 系統公告

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `lobby:notice:sequence` | String | Lobby (admin) | 永久 | 公告遞增序號。 |
| `lobby:notice:records` | Hash | Lobby | 最晚公告到期時間 | 以序號保存有效公告 DTO。 |

## `central:*` — Central 路由與鎖

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `central:routing:{playerID}` | Hash | Central | 24 小時（`RoutingTTL`，寫入刷新；登出刪除） | 路由表：欄位 `gate`、`world`、`account` |
| `central:account:{account}` | Hash | Central | 7 天（`AccountCacheTTL`） | 帳號快取：欄位 `uid`、`wid`、`hash`（passwordHash） |
| `central:lock:account:{account}` | String (NX+PX) | Central | 15s（`redisLockTTL`）；帳號清除 1 分鐘（`purgeLockTTL`） | 登入與帳號清除的分散式鎖，值為持鎖 token |
| `central:world:drain` | Set | Lobby (admin) | 排空期間 | 排空中的 World ID 集合 |
| `central:audit:record:{playerID}` | String | Central | 10 分鐘（`auditRecordCacheTTL`） | audit 快取記錄 |

## `requestjournal:*` — 請求去重快取

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
每筆紀錄的完整 key 為「前綴:{scope}:{requestID}」（scope 為帳號或 playerID），值為 String（JSON）。

| Key 前綴 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `requestjournal:central:login` | String | Central | 2 min（`LoginRequestCacheTTL`） | 登入請求去重（scope 為帳號；各 Central 共用） |
| `requestjournal:central:logout` | String | Central | 2 min（`LogoutRequestCacheTTL`） | 登出請求去重（各 Central 共用） |
| `requestjournal:gate:{gateID}:validate` | String | Gate | 2 min（`ValidationReplayCacheTTL`） | 恢復驗證請求去重 |
| `requestjournal:world:{worldID}:login` | String | World | 2 min（`RequestReplayCacheTTL`） | 登入請求去重 |
| `requestjournal:world:{worldID}:logout` | String | World | 2 min | 登出請求去重 |
| `requestjournal:world:{worldID}:validate` | String | World | 2 min | 恢復驗證請求去重 |
| `requestjournal:world:{worldID}:roll` | String | World | 2 min | 擲骰請求的完整回應，供重送時原樣回傳 |
| `requestjournal:world:{worldID}:upgrade` | String | World | 2 min | 建築升級請求的完整回應 |

## `fault:archive:*` — 故障歸檔

| Key 模式 | 類型 | Owner | 用途 |
|----------|------|-------|------|
| `fault:archive:*` | — | 外部工具 | 冷儲存歸檔（寫入端在 Go 程式碼之外） |
