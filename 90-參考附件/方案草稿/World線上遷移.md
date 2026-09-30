# World 線上遷移（熱更新）

> 狀態：初步設計草稿，待後續調整。
> 目標：在 client 不斷線的前提下，熱更新 World 版本（路線 B：零感知 live migration）。
> 對照：快速重啟方案（路線 A）為「同 IP:port 重啟 + 現有 30s 寬限窗恢復機制」，本文件只涵蓋路線 B。

## 概述

World 是有狀態節點（記憶體玩家態 + channels）。
熱更新 = 把線上玩家從舊 World 節點遷移到新版本節點，且 client（只連 Gate）不斷線。

本方案以 Central 為協調者，逐玩家進行線上遷移（`MigrateUserToWorld`），
玩家 TCP 不斷線，遊戲操作只在毫秒級暫停窗內延遲，進度零丟失。

## 設計前提（已從代碼查證的事實）

| 事實 | 出處 |
|---|---|
| client 只連 Gate，Gate 依 session／user 的 `WorldID` 維護 World 路由索引 | `cmd/gate/internal/app/session_mutation.go`、`cmd/gate/internal/app/reconcile.go` |
| Gate session 與 user 都有 `WorldID`，且有 `worldIndex/usersByWorld` 索引，reconcile 會自動收斂不一致 | `cmd/gate/internal/app/session_mutation.go`、`reconcile.go` |
| 所有 World 共用同一 GameStore MySQL（`world_db`） | `cmd/world/internal/store/gamedb/gamedb_connect.go` |
| World 間無直連，經 Central broker（已有 `ForwardToWorld` 30301） | `PJA-Server/pkg/protocol/s2s_message_id.go` |
| Central 已有 World drain 機制：`SADD central:world:drain 1001` 後新玩家不再派入 | `cmd/central/internal/app/world_drain.go`、`world_selection.go` |
| Central 路由有三處：memory（`users/usersByWorld`）、Redis 路由表、MySQL `user.world_id` | `cmd/central/internal/app/user_routing_*` |
| Client 訊息 ID 使用 PJA-Protocol canonical ledger；S2S 訊息 ID 由 Server 維護：Auth 30001-30011、System 30101-30104、Control 30301-30302、Audit 30401-30402 | `PJA-Protocol/Datas/__enums__.xlsx`、`PJA-Server/pkg/protocol/s2s_message_id.go` |
| 冪等基礎設施：`request_id` journal（scope 去重/回放） | `pkg/requestjournal` |

**核心洞察**：玩家消息在 Gate 依 `user.WorldID` 轉發。只要在正確的同步屏障下把
「Gate 的 WorldID、Central 路由、MySQL 持久 world_id、兩個 World 的記憶體態」一致地切到新節點，
玩家 TCP 不斷線，遊戲操作只在毫秒級暫停窗內延遲。

## 總覽

- **遷移單元**：單一玩家（避免整批切換的爆炸半徑）。
- **協調者**：Central（持有全局路由、是 World 間 broker、已有 drain 機制、已有 request_id journal）。
- **狀態機**（每玩家）：`idle → pausing → handoff → prepared → switched → committed → done`，任一步失敗 `→ rollback → idle`。
- **同步屏障**：舊 World「暫停 + flush 完成」→ 新 World「載入完成」→ Gate「切換」→ 舊 World「清理」（最後一步，保證隨時可回滾）。

## 單玩家遷移流程

前置：新 World 1002 已上線（連 Central、註冊 lobby）；1001 已 `SADD central:world:drain 1001`（擋新玩家）；玩家 `logged_in`、Gate 在線。

1. **觸發**：運維腳本/API 送 `MigrateUserToWorld`（帶 `request_id` + `trace_id`）給 Central。
2. **Central → Gate：`MigratePause`**（playerID, src=1001, dst=1002）
   - Gate：鎖定該玩家轉發，後續玩家消息進本地 buffer（心跳不受影響）；回 `MigratePauseAck`。
   - 作用：舊 World 不再收到該玩家新消息，在途請求自然排空。
3. **Central → 舊 World 1001：`MigrateHandoff`**（playerID, gateID, dst=1002）
   - 1001：標記該玩家 in-migration（收到後續消息直接丟棄/回「遷移中」）；快照記憶體態（channels 成員、暫態資料）；回 `MigrateHandoffAck`。
   - 作用：把最新進度落到共享 DB，供新 World 讀取；記憶體態（channels）走快照經 Central 轉發（保持「無 World 直連」原則，payload 小）。
4. **Central → 新 World 1002：`MigratePrepare`**（playerID, gateID, src=1001, 快照）
   - 1002：建立 `user` 記錄綁定 gateID；依快照 join channels；回 `MigratePrepareAck`。
   - 作用：新 World 就緒，但玩家仍未被指到它（雙寫窗口不存在）。
5. **Central → Gate：`MigrateSwitch`**（playerID, newWorld=1002）
   - Gate：原子更新 `session.WorldID` + `user.WorldID` → 1002（同步維護 `worldIndex/usersByWorld` 索引）；恢復轉發，buffer 按序發往 1002；回 `MigrateSwitchAck`。
   - 作用：**切換瞬間**。此後玩家所有新消息直接進新 World。
6. **Central 提交（關鍵路徑，同步）**：
   - memory 路由：`user.WorldID=1002`、`usersByWorld`/`usersByGate` 索引遷移；
   - MySQL `user.world_id=1002`（**同步寫，失敗即回滾整個遷移**，避免下次登入指回 1001）；
   - Redis 路由表更新（`user_routing_redis_write`）。
7. **Central → 舊 World 1001：`MigrateComplete`**
   - 1001：清理該玩家（退出 channels、清 Redis player store 殘留）；回 Ack。
   - 作用：**至此舊 World 才放棄該玩家**；此步之前回滾隨時可行。
8. **收尾**：1002 該玩家正式 active。1001 全部玩家遷完後：優雅關機（`WaitPersist` + `FlushPlayerStates`）→ 部署新版本 → 清空 drain 集合、新版本以原 1001 或新 ID 接回。

## 資料一致性保證

- **進度零丟失**：暫停 → 排空 → 舊 World 將記憶體態 flush 到共享 `world_db` → 新 World 才載入。串行屏障保證新 World 讀到的是含最後一條已處理消息的進度。
- **無亂序**：暫停期間消息 buffer 在 Gate，切換後按原序發往新 World。
- **冪等**：`request_id` journal，`scope=migrate:<playerID>`，超時重試只回放結果，不重複遷移。
- **回滾安全**：`MigrateComplete`（第 7 步）之前，舊 World 保有完整玩家態；任一步失敗 → Gate 切回 1001、1001 解除暫停、1002 清理該玩家，玩家無感（最多一次 flush 已寫入 DB，無損失）。

## 競態處理

- **玩家登出/斷線**：登出優先——Gate 中止 buffer、放棄遷移，走正常登出鏈路（Central logout 以 `playerID+request_id` 去重，天然防雙跑）。
- **遷移中 Gate/World 斷線**：沿用現有 30s grace 機制；斷線即中止遷移回滾；恢復後由既有 `ValidateWorldToGate`/reconcile 收斂。
- **玩家下線重登**：以 MySQL `world_id` 為準（遷移成功才寫），天然一致。
- **重複觸發**：`request_id` 去重擋掉。

## 新增訊息（按 S2S 分區編號）

- Gate↔Central：`MigratePause` / `MigrateSwitch` / `MigrateRollback`（30501-30503 區段）+ Ack
- Central→World：`MigrateHandoff` / `MigratePrepare` / `MigrateComplete`（30601-30603 區段）+ Ack
- World→Central：遷移 Ack（30701-30703 區段）

## 批量熱更新編排

1. `SADD central:world:drain 1001` → 擋新玩家；
2. 啟動 1002（新版本）→ 確認 lobby 心跳、Central 連線；
3. 腳本遍歷 1001 在線玩家，**限流**（如並發 100）逐個觸發遷移；
4. 1001 清空 → 優雅關機 → 1002 正式承接（或下線後以原 ID 復用）；
5. 監控：遷移成功/失敗/耗時、Gate buffer 深度、1001→1002 負載曲線；遷移後跑 Central 既有三方 audit 對帳。

## 誠實的局限

- 嚴格說這是「近零感知」：暫停窗（毫秒~秒級）內玩家操作有延遲，但**不斷線、不丟進度**。真正協議級零感知需雙寫+去重緩衝，複雜度陡增，一般遊戲營運可接受前者。
- 遊戲操作本身需冪等（舊 World 最後一條消息的副作用已進 DB，新 World 載入即含之，無重複執行）。

## 分層實施計劃（待調整）

1. **Phase 1：遷移協定與訊息骨架**
   - 新增 6 組遷移訊息 + Ack，於 `pkg/enum/msg_id_*` 按分區註冊
   - Central 玩家遷移狀態機（`idle→…→done/rollback`）+ `request_id` journal（`scope=migrate:<playerID>`）

2. **Phase 2：World 側遷移支援**
   - 舊 World：`MigrateHandoff` handler（暫停玩家、快照 channels、回 Ack）
   - 新 World：`MigratePrepare` handler（建 user 綁 gateID、join channels）
   - `MigrateComplete` 清理與 `MigrateRollback` 復原

3. **Phase 3：Gate 側遷移支援**
   - `MigratePause`：玩家轉發 buffer（暫停→緩衝→恢復）
   - `MigrateSwitch`：session/user 的 WorldID 原子切換 + `worldIndex/usersByWorld` 索引更新
   - `MigrateRollback`

4. **Phase 4：Central 協調與路由提交**
   - 遷移狀態機時序控制、失敗回滾、登出/斷線競態
   - memory 路由 + MySQL `user.world_id` + Redis 路由表三處一致更新

5. **Phase 5：測試**
   - 各 handler 單元測試（狀態機、buffer、索引切換）
   - 整合測試：遷移全流程、失敗注入、登出/斷線競態、遷移後 audit 對帳

6. **Phase 6：編排與觀測**
   - `scripts/` 批量遷移工具（drain → 限流遷移 → 清空下線）
   - 遷移指標（成功/失敗/耗時/buffer 深度）+ 文件更新（`docs/`）

## 相關文件

- [[系統導讀]]
- [[連線與登入流程]]
- [[資料一致性]]
- [[World指派策略]]（「擴展方向：玩家遷移 / MigrateUserToWorld」為本方案的起點）
- [[模組關係]]（訊息 ID 分區）
