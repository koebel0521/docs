---
tags: [refactor, solid, dry, kiss, review]
created: 2026-10-03
---

# SOLID／DRY／KISS 審查與重構紀錄

本文件記錄 2026-10-03 對 PJA-Server 做的一次原則審查與後續重構。屬歷史紀錄，不作為現行規範；現行行為以代碼為準。

## 結論

- 沒有嚴重的 SOLID 結構問題。介面幾乎都由消費端定義，沒有「一個結構配一個介面」的過度抽象。
- 主要問題是 **DRY**：同一套規則或樣板被複製多份，規則改一處、另一處就不同步。其次是少數肥大的 struct 與函式（SRP）。
- 審查以代理分區讀碼（`cmd/world`、其他 `cmd`、`pkg` 與 `internal`、`tools`），只看 SOLID、DRY、KISS，不含邊界、效能與安全。
- 審查結論有數項被後續讀碼推翻或證實收益太小，已跳過，見「沒做的項目」。**審查清單不等於待辦清單**，動手前要重新讀碼確認。

## 已完成（14 個 commit，基準 `03dede23d`）

### 階段 0：死碼與重複的小工具

| commit | 內容 |
| --- | --- |
| `780a5b0b8` | 刪除 `dice_token` 永遠走不到的 refund 參數與分支（連同僅測試用的 helper）；gate 的 `remoteIP`／`extractIP` 合併為 `flowbase.RemoteIP`；gate user event 的 `Option` 改為 `func(*Event)` |

### 階段 1：共用結算規則（價值最高）

記憶體版 GameStore、MySQL 版 GameStore、`tools/boardsimulator` 各有一份工廠子彈結算，錯誤訊息也相同。

| commit | 內容 |
| --- | --- |
| `9781a23be` | `internal/game/board` 新增 `FactoryBulletQuantity`、`FactoryBulletCost` 純函式與表驅動測試；gamestore 兩個 store 改呼叫共用函式；抽出 `planBuildingUpgrade`、`buildingEffectRows`、`researchEffectRows`、`validateEventItemReward` |
| `1b41276f5` | `boardsimulator` 的 `settleFactory` 改呼叫共用函式 |

要點：

- 兩個 store 保留各自的 I/O 順序（例如 MySQL 版先載入金幣列才檢查成本溢位），只共用驗證與計算。
- 模擬器的金幣是變數、World 的金幣是道具，資料模型不同，所以建築升級的扣費流程沒有硬合併。

### 階段 2：GameStore 與型別斷言

| commit | 內容 |
| --- | --- |
| `2769144de` | `playerTx` 的 20 個暫存欄位與 `PlayerChanges` 一一對應，改為持有一個 `PlayerChanges`（原本內嵌，後依技能審查改為具名欄位 `changes`）；`commit` 拆成 `applyToBase` 與 `mergePlayerChanges`；新增反射測試，確保 `PlayerChanges` 新增欄位時 `mergePlayerChanges` 不會漏合併而靜默丟資料 |
| `13828daee` | `variable.NewWithMigrations` 改收 `MigratingStore`；`app` 的常駐資料管理只在 `SetGameStore` 判斷一次；`ConfigVersionPlayerInitializer` 併入 `PlayerInitializer`。缺少能力改由編譯器檢查。`TopologyMigrationPersistence` 原本也併入 `Persistence`，後依技能審查還原為獨立的小介面，並加 `var _ TopologyMigrationPersistence = gamestore.GameStore(nil)` 讓缺少方法時編譯期失敗 |
| `399c9a898` | 依技能審查修正：`playerTx` 改用具名欄位 `changes`；`TopologyMigrationPersistence` 還原為獨立小介面並加編譯期檢查；以拋棄式 MySQL 容器補跑整合測試 |

### 階段 3：服務骨架

| commit | 內容 |
| --- | --- |
| `32abf2ddf` | `ShutdownOrchestrator.RunApp`：啟動自檢、執行、關機的共用尾段，chat、user、world、central、gate 五個服務改用；chat 自檢失敗時也會先 `Stop` |
| `509acbc36` | `pkg/mysql.ReadyChecker`：四個服務原本寫法不一致（只有 record 有逾時），統一為 3 秒逾時 |
| `17e74b683` | 三個服務相同的 heartbeat handler 合併為 `msgsys.HandleHeartbeatReq` |

### 階段 4：肥大函式

| commit | 內容 |
| --- | --- |
| `e46092264` | `board.roll()` 的格子 switch 抽成 `appendSpaceEvents`；兩處重複的研究所重置事件合併；隨機事件的加速與隨機道路加星分支抽成獨立方法。`BenchmarkRollDiceWithProgress` 前後在雜訊範圍內，記憶體配置完全相同（9 allocs／1264 B） |
| `fa831aa7c` | 擲骰回應的映射函式（約 130 行）從 `roll_dice.go` 移到 `roll_dice_protocol.go`；mission 的 `activation`、`progress` 移除立即呼叫的閉包 |

`board.roll()` 沒有改成查表：專案效能優先，enum 上的 `switch` 本來就慣用。

### 階段 5：小型 DRY 與 tools

| commit | 內容 |
| --- | --- |
| `74e2181c5` | `pkg/redis` 抽 `formatAll`、`zaddCmd`、`expireCmd` |
| `592478781` | `consistencysnapshot` 改用 `diag.WriteResult`；新增 `cmd/world/tools/internal/toolcli`，三個 World 維運工具改用 |
| `13edf9365` | lobby 的 `CentralInfo`、`WorldInfo` 合併為 `ServiceInfo`（型別別名保留原名） |

## 行為變動（不是純搬動的部分）

- chat 啟動自檢失敗時，現在也會先呼叫 `Stop`，與 central、gate 一致（`32abf2ddf`）。
- chat、central、world 的 MySQL ready checker 原本沒有逾時，現在 `Ping` 最多等 3 秒（`509acbc36`）。
- `consistencysnapshot` 以 stdout 輸出且 `-fail-on-endpoint-down` 時，現在先印出快照再回傳失敗；原本檢查失敗就不印（`592478781`）。
- 模擬器的工廠結算順序與記憶體版對齊：先算產量再算金幣。「金幣不足」與「子彈數溢位」同時發生時，模擬器現在回傳錯誤，以前是靜默不生產。要讓子彈數超過 `MaxInt32` 需要儲存上限設定大於 `MaxInt32`，實務上不會發生。
- mission 的 `activation`、`progress` 外層錯誤前綴 `execute mission activation:`、`execute mission progress:` 移除，內層訊息保留。

## 沒做的項目與原因

審查原本列出、後來讀碼後決定不做：

| 項目 | 原因 |
| --- | --- |
| `RollDiceDeps.handle` 包辦所有事 | 讀完整檔後，`handle` 只有 36 行，已拆成 `roll`、`save`、`saveWithLotteries` 等，審查說過頭 |
| `GameStore` 38 方法拆角色介面 | 消費端已用自己的小介面；拆生產端要重寫斷路器與記憶體版的全部轉發，風險高、收益主要是少改幾個檔 |
| 斷路器 `run` 去重 | gamestore 與 playerstore 的 `call` 錯誤處理不同（playerstore 有 metrics 與 `ErrCircuitOpen` 分流），只有 3 行的 `run` 相同，抽出去得不償失 |
| `itemaction` Executor 註冊機制 | 只有 `adjust_variable` 一種動作，但 `services`、`board` 的測試廣泛注入 stub handler，要改 20 多處測試，不是零風險清理 |
| `effect`、`shop` 的 reflect typed-nil 檢查 | 既有測試 `TestNewRejectsNilStore/typed_nil` 明確要求拒絕，是刻意行為 |
| `dataconfig/snapshot.go` 任務規則外移 | `MissionProgressCompatible` 在 snapshot 驗證內部就被呼叫，要搬出去得反向注入 callback；`reloadableFiles` 是資料表，不是 OCP 問題 |
| `itemIDs`／`effectIDs`／`eventIDs` 抽泛型 | 三個各 11 行，抽出來每個仍約 6 行加一個 helper，淨省不了幾行 |
| `flowbase` getter 與每請求建 `Coordinator` | 動 5 個檔、約 20 處呼叫，不屬於零風險清理 |
| `Multi` 的未用 `ctx` 參數 | 簽名改動波及所有呼叫端 |
| world、gate、central 的 `App` God struct 拆分 | 沒有具體缺陷要修；只有 gate 的 `InsertUser` 持有兩把鎖再呼叫會取第三把鎖的函式，靠註解維持鎖序，是真風險。要做建議單獨針對 gate 的 channel 投影與鎖序，先補並發測試 |
| `GoBackground`／`WaitBackground` 上移到 `AppBase` | central 用 `*sync.WaitGroup` 欄位、world 有 nil receiver 保護、gate 直接暴露 `BackgroundWg`，三者語意不同，要先讀完所有呼叫端 |
| `lobbyreporter.Start`、`tcpserver` 預設 option、`startup.Bootstrap` | 各服務前置段的 option 差異多（規則、reload、debug port、peer 列表），抽出來要帶很多旋鈕 |
| `mission` 改傳 `Deps`／函式取代 `*worldboard.Adapter` | `HandleClaim` 10 個位置參數確實難讀，但牽涉 5 個檔案簽名與大量測試，留到動 mission 時再做 |
| tools 的死信掃描與 `rediscli.Snapshot` 去重 | 三個工具輸出結構不同，收益小於風險 |
| `internal/game/*config` 共用 `requireTable` 與 sentinel | 各套件 `ErrInvalidConfig` 同名不同值，合併會動到錯誤判斷，沒有明確缺陷 |
| gate `on_chat.go` guard、lobby admin handler 樣板 | 沒讀過全部呼叫端與測試 |
| `pkg/msg/**` 58 個訊息的樣板 | 性能優先的取捨；要改應改用 codegen，會動到協議產生流程 |

## 技能審查（以 `golang-*` 技能為準）

事後對照 `golang-refactoring`、`golang-design-patterns`、`golang-structs-interfaces`、`golang-error-handling` 四個技能，發現下列出入：

| 出入 | 處理 |
| --- | --- |
| `golang-refactoring` 規定結構與行為改動不得放同一個 commit。`32abf2ddf`（chat 自檢失敗先 `Stop`）、`509acbc36`（ready checker 加逾時）、`592478781`（stdout 輸出順序）、`fa831aa7c`（搬移函式同時移除 mission 外層錯誤前綴）違反 | 未改寫歷史；上方「行為變動」清單已逐項列出。之後的重構要把行為變動拆成獨立 commit |
| `golang-refactoring` 要求有安全網才改。MySQL 路徑改動時整合測試沒跑 | 已補跑，見「驗證」 |
| `golang-refactoring` 要求優先用 gopls Rename，不要手改。階段 2 的欄位改名用正則批次替換 | 以編譯與測試驗證，結果正確；之後同類改名改用 gopls |
| `golang-structs-interfaces`：只在內部使用時應用具名欄位，不要內嵌。`playerTx` 內嵌 `*PlayerChanges` 讓 `Empty()` 與 `PlayerID` 變成公開成員 | 已改為具名欄位 `changes` |
| `golang-structs-interfaces`：介面保持 1 到 3 個方法，並認可對可選小介面做 comma-ok 斷言。把 `TopologyMigrationPersistence` 併入 `Persistence`（5 個變 6 個方法）違反 | 已還原，並加編譯期檢查 |

與專案規範的衝突，以專案規範為準的項目：

- 日誌庫：技能建議 `slog`，但 `CLAUDE.md` 硬規則是 zerolog 為唯一日誌庫，維持 zerolog。
- Functional Options：技能只在「驗證可能失敗」時才要求 `Option` 回傳 `error`；[工作規範](../../00-導讀/工作規範.md)寫得較嚴。階段 0 把 gate user event 的 `Option` 改成 `func(*Event)`（它只賦值、不會失敗）符合技能，保留。

另外，工作規範的「漸進式優化」要求只在正在修改的區域做優化。這次是受託全專案審查後的整批重構，分成 14 個小 commit，每個都可獨立回滾，但範圍超出該條的預設。

## 驗證

- 每個階段都跑過 `go build ./...`、`go vet` 與受影響範圍的測試；階段 5 結束後全專案（`cmd`、`pkg`、`internal`、`tools`）測試通過。
- 每個 commit 都過 pre-commit（golangci-lint、goimports）。過程中被 `wrapcheck`、`exhaustive`、`gocyclo`、`gosec` 擋下的地方，都以有理由的 `//nolint` 或小幅重構處理。
- **整合測試**：整合測試檔有 `//go:build integration`，沒加 `-tags integration` 時根本不會編譯，`go test` 只回 `ok`，不會顯示跳過。事後用拋棄式 `mysql:8.4` 容器補跑：`cmd/world/internal/store/gamestore` 的 MySQL 版與 `GAMESTORE_IMPL=memory` 的記憶體版各 150 個整合測試全過（含 `FactoryBulletCostOverflow`、`FactoryBulletQuantityOverflow`、`UpgradeBoardBuilding*`），`cmd/world/...` 與 `internal/...` 加 tag 後全過。這包含技能審查後的具名欄位與介面還原。

## 未驗證

- **沒有實際啟動服務驗證**：階段 3 的 `RunApp` 關機流程只有單元測試。建議本機啟動全部服務後按 Ctrl+C，確認關機順序。
- `./integration/...`（需要完整服務堆疊）沒有跑。

## 後續建議

1. 先補 gate `App` 的並發測試，再處理 channel 投影與鎖序。
2. 動 mission 時一併把 `HandleClaim` 的位置參數改成 `Deps`。
3. 之後再做審查，先讀碼確認再列清單；本次有約三分之一的項目在讀碼後被判定說過頭或收益太小。
