# World 指派策略

本文件說明 Central 在登入流程中如何為玩家選擇 World 節點。

## 概述

Central 在處理 `LoginGateToCentral` 時，根據玩家是否已有帳號記錄採取不同策略：

- **既有玩家**（DB 有記錄）：沿用資料庫中的 `WorldID`，不重新指派。
- **新玩家**（DB 無記錄）：由 `selectNewUserWorld`（`cmd/central/internal/flow/login/route.go`）選擇 World：先看 canary 流量切片，沒命中再依負載選擇。

## 指派演算法

### Canary 流量切片

`traffic_slice_percent`（0 停用）與 canary World 由 `App.SetTrafficSlice(percent, worldID)` 設定（啟動與熱重載都經 `cmd/central/traffic_slice_config.go`）。`App.TrafficSliceWorldID(account)` 以帳號的 FNV-1a 雜湊對 100 取餘數，小於 `percent` 的新帳號導向 canary World；canary World 也必須可選（未排空、不在重連冷卻、socket 存在），否則退回下面的最少負載選擇。只影響新玩家，既有玩家仍回資料庫記錄的 World。

### 最少負載優先

沒有命中流量切片時使用的預設策略：選擇邏輯在 `cmd/central/internal/app/world_registry.go` 的 `App.LeastLoadedWorldID`，由登入流程 `cmd/central/internal/flow/login/route.go` 呼叫。

流程：

1. 呼叫 `App.LeastLoadedWorldID()`，遍歷所有已連線的 World，選擇 `usersByWorld[worldID]` 數量最少的。
2. 會先跳過正在 drain，或剛恢復仍在 `worldRouteCooldown` 冷卻窗內的 World。
3. 若該 World 的 socket 仍然存活，直接使用。
4. 沒有可選 World（全部排空、冷卻、未連線，或 socket 已斷線）時回傳 nil，Central 拒絕這次新登入／新註冊並回可重試的 `AuthServiceUnavailable`（不快取，Gate 以同一 `RequestID` 重試會重新選 World）。不再退回隨機，避免新玩家被導向排空中的 World；既有玩家依已存路由重登，不經過這個選擇器。

```
LeastLoadedWorldID()
├── 遍歷 connectedWorlds map
├── 比較 len(usersByWorld[worldID])
└── 回傳玩家數最少的 WorldID（無連線則回傳 0）
```

## 限制與已知問題

### 1. 負載計算僅基於 Central 記憶體

`usersByWorld` 統計的是 Central 記憶體中登記的玩家數，而非 World 節點回報的實際負載。
若 Central 重啟，記憶體清空，所有 World 的負載都會被視為 0，直到玩家重新登入。

### 2. 不支援加權或容量上限

目前沒有每個 World 的最大容量設定。若某些 World 節點硬體規格不同，
無法透過權重反映差異，只能依賴玩家數的自然平衡。

### 3. 既有玩家不會遷移

既有玩家永遠回到資料庫記錄的 WorldID。即使該 World 嚴重過載，
也不會自動遷移到負載較低的節點。遷移需要手動變更 DB 記錄：

1. 確認玩家離線（Central 記憶體沒有該玩家）。
2. 更新 `user.world_id`。
3. 執行 `go run ./tools/private/pja staterepair -clear-route -account <帳號> -confirm`，清掉 Central 的帳號快取與路由。

登入先查 Central 記憶體與 Redis 帳號快取（`central:account:{account}`，TTL 7 天）才回查 MySQL；
略過第 3 步時，快取過期前登入都會導向舊 World。

### 4. 剛恢復的 World 會先冷卻

剛重新連線的 World 會先進 `worldRouteCooldown`，避免恢復瞬間立刻吃新流量。
這樣可以先觀察節點是否穩定，再慢慢放量。

### 5. 單一 Central 視角

多 Central 架構下，每個 Central 只看到自己記憶體中的玩家分佈，
可能導致各 Central 對「最少負載」的判斷不一致。

## 擴展方向

| 需求 | 可能做法 |
|------|---------|
| World 容量上限 | `LeastLoadedWorldID` 中加入 `maxPlayers` 檢查，滿員時跳過 |
| 加權指派 | World 定期回報 CPU/記憶體負載，Central 綜合考量 |
| 玩家遷移 | 新增 `MigrateUserToWorld` 訊息，支援線上搬移 |
| 多 Central 一致性 | 透過 Redis 共享各 World 的即時玩家數 |

## 相關文件

- [[連線與登入流程]]
  - 登入流程中的 World 指派會用到這份策略
- [[資料一致性]]
  - 說明 WorldID 與路由資料的來源與一致性
