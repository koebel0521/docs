# Redis Key Schema 總表

本文件列出專案所有 Redis key prefix 與其 owner。TTL 以程式常數為準（括號內為常數名稱）；Redis TTL 一律以 PEXPIRE／SET PX 毫秒精度設定。

## `lobby:*` — 服務探索（lobbyreporter）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `lobby:gate:{gateID}` | Hash | Gate | 25s（`keyTTL`，每次回報刷新） | Gate 地址與負載；Gate 在玩家 listener 啟動之後才開始公告 |
| `lobby:central:{centralID}` | Hash | Central | 25s（`keyTTL`） | Central 地址；Central 在 S2S 設定驗證與 TCP 服務啟動之後才開始公告，S2S 設定錯誤時不公告 |
| `lobby:world:{worldID}` | Hash | World | 25s（`keyTTL`） | World 地址與容量；World 在載入設定、遷移、啟動服務且啟動自檢通過之後才開始公告（第十二輪起，自檢失敗直接退出） |
| `lobby:gates`、`lobby:centrals`、`lobby:worlds` | Set | 各服務 | 無 | 各類服務的 ID 索引（`IndexKey`），成員為服務 ID；服務停止時移除 |

Gate、Central、World 每輪公告前先跑就緒檢查（`lobbyreporter.New` 的第四個必要參數，接 `debug.Server.CheckServing`；nil 時 panic，原 `WithReadyCheck` Option 已移除），與 `/ready` 是同一組檢查（含 runtime-only 的啟動閘門與 `time_sync`），`/ready` 503 時就不公告。第十一輪接的是 `CheckReady`（第十二輪改名 `CheckStartup`），只跑啟動自檢子集，`/ready` 503 時仍會公告：連不上 Central 的 Gate 卻以 0 負載排在 Lobby 清單最前，所有登入都回 `AuthServiceUnavailable`（第十二輪 N-M1 修正）。取捨：Central 全斷時所有 Gate／World 都會撤除公告；重啟後第一次公告要等閘門放行且收到 TimeSync 後的下一輪（最多 10 秒）。不就緒時刪除自己的 key 與索引成員而非續期，恢復後重新公告，只在狀態轉變時記日誌（Warn `lobby reporter: not ready, withdrawing announcement`／Info `lobby reporter: ready, announcing`；啟動後第一次未就緒不記 Warn）；服務關機（ctx 已取消）時不跑檢查，不會誤記未就緒。Redis、MySQL 或 `data_config` 檢查短暫失敗時，該實例也會暫時從 Lobby 清單消失。原本監聽失敗或 accept 迴圈 panic 後仍每 10 秒續期，負載歸零的 Gate 排在 Lobby 清單最前，把新玩家導向連不上的位址。

## `player:*` / `world:*` — 玩家狀態（playerstoreredis）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `player:{playerID}` | String (JSON) | World | 24 小時（`playerTTL`，每次寫入刷新） | 玩家登入狀態快照（gateID, worldID） |
| `world:{worldID}:players` | Set | World | 無 | World 上所有玩家的索引（供 restore）；刪除或改存到其他 World 時由 Lua 腳本移除成員，restore 時清掉過期成員 |

## `gm:*` — GM 操作

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `gm:ban:{account}` | String | Lobby (admin) | 封禁期間（`duration` 參數，預設 30 天 `defaultBanTTL`） | 帳號封禁標記（key 存在 = 被封禁） |
| `gm:kick:{gateID}` | List | Lobby (admin) → Gate | 消費後移除 | GM 踢人佇列，Gate 用 BLPOP 消費（BLPOP 出錯後以 1 秒起倍增到 30 秒、±30% 抖動的指數退避重試，成功即歸零，Redis 長時間故障時不再每秒一行 Warn，多台 Gate 也不會同一秒打回）；playerID 解析失敗記 Warn 並計 `game_async_failure_total{queue="kick",outcome="invalid_payload"}` |
| `gm:version:policy` | String（JSON） | Lobby (admin) → Gate | 永久 | Client 版本政策：維護旗標、線上玩家寬限期與各平台最低／最新遊戲版本；Gate 每 10 秒重新載入，讀取失敗沿用舊值 |
| `gm:grant_item:{worldID}` | List | Lobby (admin) → World | 消費後移除 | GM 發道具佇列，World 用 BRPopLPUSH 消費 |
| `gm:grant_item:processing:{worldID}` | List | World | 消費期間 | 處理中指令（BRPOPLPUSH 從佇列移入，處理完移除，避免重複消費）。多 worker 時第 N（N≥1）個 worker 用 `gm:grant_item:processing:{worldID}:{index}`，第 0 個沿用不帶 index 的 key；調降 worker 數後殘留的 `:{index}` 清單要手動檢查，不會自動回收 |
| `gm:grant_item:retry:{worldID}` | ZSet | World | 重試期間 | 失敗指令重試佇列，按時間排程 |
| `gm:grant_item:capability:{worldID}` | String | World | 永久（on/off） | World 是否支援 GM 發道具 |
| `gm:grant_item:deadletter:{worldID}` | List | World | 7 天（`grantitem.DeadLetterTTL`；掛在整個 list 上，每次 push 都重設，所以是「最後一筆寫入後 7 天」，不是每筆各自 7 天） | 超過重試次數的死信佇列；長度上限 10000（`grantitem.DeadLetterMaxLen`），push 後超過就 `LTRIM` 裁掉最舊項目（裁剪失敗只記警告） |

佇列中的指令是 `grantitem.Command` 的 JSON，World 以嚴格模式解碼（拒絕未知欄位與尾部資料），無法解碼的指令原文放進死信保留 7 天。`Command` 新增欄位時必須先升級所有 World，再升級寫入佇列的 Lobby；反過來升級時，舊 World 會把新格式的指令全部放進死信，升級 World 與工具後再以 `tools/private/deadletterreplay` 重送（舊版工具同樣無法解碼，會直接報錯）；`tools/private/datasurgeon -mode deadletter` 可列出或清除無法解碼的項目。

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
| `central:lock:account:{account}` | String (NX+PX) | Central | 15s（`redisLockTTL`）；帳號清除 1 分鐘（`purgeLockTTL`） | 登入與帳號清除的分散式鎖，值為持鎖 token。取鎖的用戶端逾時（`DeadlineExceeded`）不代表 `SET NX PX` 沒在 Redis 執行，鎖可能已以本次 token 寫入卻沒人會解；逾時時會用脫離取消的 ctx 對同一 token 補一次 `Unlock`（比對 token，不會誤刪別人的鎖），同帳號登入不必等滿 15 秒。登入取鎖後以 `LoginAccountLockHold`（`redisLockTTL` 減 5 秒）作為處理期限，逾時即中止，不讓鎖在流程中途過期 |
| `central:world:drain` | Set | Lobby (admin) | 排空期間 | 排空中的 World ID 集合 |
| `central:audit:record:{playerID}` | Hash | Central | 10 分鐘（`auditRecordCacheTTL`） | audit 快取記錄：欄位 `account`、`worldID`、`hash`（passwordHash）；用 `HGETALL` 查，`GET` 會得到 WRONGTYPE |

## `combat:*` — PvP 候選索引（combatstore）

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| `combat:eligible-player` | Set | World（所有 World 共用） | 無 | 可配對玩家 ID 候選集合；Redis 重啟或 `FLUSHALL` 後由玩家操作時重寫補回，或重啟 World 觸發重建 |
| `combat:eligible-player:rebuild:{nanoBase36}` | Set | World | 1 小時 | 重建時先寫入的專屬暫存 key，完成後以 Lua 原子換上正式集合；正常不會殘留，崩潰時殘留靠 TTL 自行清除 |

## `requestjournal:*` — 請求去重快取

| Key 模式 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
2026-10-30 起 Central、Gate 與 World 的登入、登出、恢復驗證 journal 已刪，不再寫入新鍵；下列已刪除者的舊鍵最多殘留一個 TTL（2 分鐘），`rediskeys.GateValidateJournal*` 保留供維運工具清除舊鍵。

每筆紀錄的完整 key 為「前綴:{scope}:{requestID}」（scope 為帳號或 playerID），值為 String（JSON）。

| Key 前綴 | 類型 | Owner | TTL | 用途 |
|----------|------|-------|-----|------|
| ~~`requestjournal:central:login`~~ | String | Central | — | 已刪除（2026-10-30）：登入請求去重（scope 為帳號；各 Central 共用） |
| ~~`requestjournal:central:logout`~~ | String | Central | — | 已刪除（2026-10-30）：登出請求去重（各 Central 共用） |
| ~~`requestjournal:gate:{gateID}:validate`~~ | String | Gate | — | 已刪除（2026-10-30）：恢復驗證請求去重 |
| ~~`requestjournal:world:{worldID}:login`~~ | String | World | — | 已刪除（2026-10-30）：登入請求去重 |
| ~~`requestjournal:world:{worldID}:logout`~~ | String | World | — | 已刪除（2026-10-30）：登出請求去重 |
| ~~`requestjournal:world:{worldID}:validate`~~ | — | World | — | 已停用：World 恢復驗證去重只存本機（requestID 由本行程以時間單調產生，重啟後不重複），不寫 Redis |
| `requestjournal:world:{worldID}:roll` | String | World | 2 min | 擲骰請求的完整回應，供重送時原樣回傳；值為 `0x00` ＋ 二進位（2 byte 大端 MessageID ＋ payload），讀取相容舊版 JSON |
| `requestjournal:world:{worldID}:upgrade` | String | World | 2 min | 建築升級請求的完整回應；編碼同 `:roll` |

## `fault:archive:*` — 故障歸檔

| Key 模式 | 類型 | Owner | 用途 |
|----------|------|-------|------|
| `fault:archive:*` | — | 外部工具 | 冷儲存歸檔（寫入端在 Go 程式碼之外） |
