# World 指派策略

本文件說明 Central 在登入流程中如何為玩家選擇 World 節點。

## 概述

Central 在處理 `LoginGateToCentral` 時，根據玩家是否已有帳號記錄採取不同策略：

- **既有玩家**（DB 有記錄）：沿用資料庫中的 `WorldID`，不重新指派。
- **新玩家**（DB 無記錄）：透過 `worldAssigner` 介面選擇 World。

## 指派演算法

### 最少負載優先（leastLoadedWorldAssigner）

這是目前的預設策略，定義於 `cmd/central/internal/handler/gate/world_assigner.go`。

流程：

1. 呼叫 `App.LeastLoadedWorldID()`，遍歷所有已連線的 World，選擇 `usersByWorld[worldID]` 數量最少的。
2. 會先跳過正在 drain，或剛恢復仍在 `WorldRouteCooldown` 冷卻窗內的 World。
3. 若該 World 的 socket 仍然存活，直接使用。
4. 若 socket 已斷線（返回 nil），fallback 到隨機選擇（`GetRandom()`）。

```
LeastLoadedWorldID()
├── 遍歷 connectedWorlds map
├── 比較 len(usersByWorld[worldID])
└── 回傳玩家數最少的 WorldID（無連線則回傳 0）
```

### 隨機選擇（randomWorldAssigner）

Fallback 策略，當 `leastLoadedWorldAssigner` 不可用或最佳 World 已斷線時使用。
直接呼叫 `worldSvc.GetRandom()` 隨機取一個已連線的 World socket。

## 限制與已知問題

### 1. 負載計算僅基於 Central 記憶體

`usersByWorld` 統計的是 Central 記憶體中登記的玩家數，而非 World 節點回報的實際負載。
若 Central 重啟，記憶體清空，所有 World 的負載都會被視為 0，直到玩家重新登入。

### 2. 不支援加權或容量上限

目前沒有每個 World 的最大容量設定。若某些 World 節點硬體規格不同，
無法透過權重反映差異，只能依賴玩家數的自然平衡。

### 3. 既有玩家不會遷移

既有玩家永遠回到資料庫記錄的 WorldID。即使該 World 嚴重過載，
也不會自動遷移到負載較低的節點。遷移需要手動變更 DB 記錄。

### 4. 剛恢復的 World 會先冷卻

剛重新連線的 World 會先進 `WorldRouteCooldown`，避免恢復瞬間立刻吃新流量。
這樣可以先觀察節點是否穩定，再慢慢放量。

### 5. 單一 Central 視角

多 Central 架構下，每個 Central 只看到自己記憶體中的玩家分佈，
可能導致各 Central 對「最少負載」的判斷不一致。

## 擴展方向

| 需求 | 可能做法 |
|------|---------|
| World 容量上限 | `worldAssigner` 中加入 `maxPlayers` 檢查，滿員時跳過 |
| 加權指派 | World 定期回報 CPU/記憶體負載，Central 綜合考量 |
| 玩家遷移 | 新增 `MigrateUserToWorld` 訊息，支援線上搬移 |
| 多 Central 一致性 | 透過 Redis 共享各 World 的即時玩家數 |

## 相關文件

- [[連線與登入流程]]
  - 登入流程中的 World 指派會用到這份策略
- [[資料一致性]]
  - 說明 WorldID 與路由資料的來源與一致性
