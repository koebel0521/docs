# Redis Key Schema 總表

本文件列出專案所有 Redis key prefix 與其 owner。

## `lobby:*` — 服務探索（lobbyreporter）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `lobby:gate:{gateID}` | Hash | Gate | 25s | Gate 地址與負載 |
| `lobby:central:{centralID}` | Hash | Central | 25s | Central 地址 |
| `lobby:world:{worldID}` | Hash | World | 25s | World 地址與容量 |

## `player:*` / `world:*` — 玩家狀態（playerstoreredis）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `player:{playerID}` | String (JSON) | World | 玩家在線期間 | 玩家登入狀態快照（gateID, worldID） |
| `world:{worldID}:players` | Set | World | 玩家在線期間 | World 上所有玩家的集合（供 restore） |

## `gm:*` — GM 操作

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `gm:ban:{account}` | String | Lobby (admin) | 封禁期間 | 帳號封禁標記（key 存在 = 被封禁） |
| `gm:kick:{gateID}` | List | Lobby (admin) → Gate | 消費後移除 | GM 踢人佇列，Gate 用 BLPOP 消費 |
| `gm:grant_item:{worldID}` | List | Lobby (admin) → World | 消費後移除 | GM 發道具佇列，World 用 BRPopLPUSH 消費 |
| `gm:grant_item:processing:{worldID}` | String | World | 消費期間 | 處理中指令（避免重複消費） |
| `gm:grant_item:retry:{worldID}` | ZSet | World | 重試期間 | 失敗指令重試佇列，按時間排程 |
| `gm:grant_item:capability:{worldID}` | String | World | 永久（on/off） | World 是否支援 GM 發道具 |
| `gm:grant_item:deadletter:{worldID}` | List | World | 7 天 | 超過重試次數的死信佇列 |

## `lobby:notice:*` — 系統公告

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `lobby:notice:sequence` | String | Lobby (admin) | 永久 | 公告遞增序號。 |
| `lobby:notice:records` | Hash | Lobby | 最晚公告到期時間 | 以序號保存有效公告 DTO。 |

## `central:*` — Central 路由與鎖

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `central:routing:{playerID}` | Hash | Central | 玩家在線期間 | 路由表：playerID → gateID + worldID |
| `central:account:{account}` | Hash | Central | 玩家在線期間 | 帳號快取：account → playerID + worldID + passwordHash |
| `central:lock:account:{account}` | String (NX+PX) | Central | 3s (PX) | 登入分散式鎖，防重入 |
| `central:world:drain` | Set | Lobby (admin) | 排空期間 | 排空中的 World ID 集合 |
| `central:audit:record:{playerID}` | String | Central | audit 期間 | audit 快取記錄 |

## `requestjournal:*` — 請求去重快取

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `requestjournal:central:{id}:login` | KV | Central | 2 min | 登入請求去重 |
| `requestjournal:central:{id}:logout` | KV | Central | 2 min | 登出請求去重 |
| `requestjournal:gate:{id}:validate` | KV | Gate | 2 min | 驗證請求去重 |
| `requestjournal:world:{id}:login` | KV | World | 2 min | 登入請求去重 |
| `requestjournal:world:{id}:logout` | KV | World | 2 min | 登出請求去重 |
| `requestjournal:world:{id}:validate` | KV | World | 2 min | 驗證請求去重 |

## `fault:archive:*` — 故障歸檔

| Key 模式 | 類型 | Owner | 用途 |
|----------|------|-------|------|
| `fault:archive:*` | — | 外部工具 | 冷儲存歸檔（寫入端在 Go 程式碼之外） |
