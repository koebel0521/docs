---
tags: [architecture, configuration, reload, deployment]
created: 2026-09-16
---

# PJA-Server bytes 配置統一與熱重載方案（設計與限制）

本文件是設計、限制與演進記錄；目前可執行的操作請先看[配置操作入口](../../30-部署與開發/配置操作入口.md)。

現行命令以[配置操作入口](../../30-部署與開發/配置操作入口.md)為準；本文件中的命令片段僅作方案脈絡與歷史參考。

## 1. 結論

PJA-Server 應統一 bytes 配置流程。

統一範圍不是只抽出 `os.ReadFile`，而是建立完整的配置版本邊界：

```text
單一版本 artifact
    ↓ 一次載入
GameConfigs
    ↓ domain 轉換與交叉驗證
不可變 Snapshot
    ↓ 驗證成功
atomic 發布給新請求
```

建議分四階段落地：

1. 統一載入，維持現有行為。
2. 建立可原子發布的版本化 artifact。
3. 只開放安全的配置表熱重載。
4. 有 migration 規則後，再擴大熱重載範圍。

Phase 2 完成前，不啟用自動 reload。

## 2. 現況與問題

### 2.1 同一批資料被重複載入

World 啟動在 [services.go](../../../cmd/world/internal/services/services.go) 內，分別載入：

- variable definitions
- effect definitions
- item definitions
- board definition
- shop quotes
- initial money

每次呼叫 generated `NewTables`。目前 `NewTables` 會讀取 11 個 bytes 表，因此同一批資料最多重複讀取與解碼：

```text
6 次 NewTables × 11 個表 = 66 次檔案讀取與解碼
```

效能不是唯一問題。六次讀取沒有共同版本邊界，更新期間可能形成：

```text
BoardTopology = 版本 A
Shop          = 版本 B
Variable      = 版本 B
GameRules     = 版本 A
```

每個 loader 單獨成功，不代表整體配置一致。

### 2.2 現有同步腳本不是原子發布

[sync-data-config.ps1](../../../scripts/sync-data-config.ps1) 目前會先刪除目標目錄，再逐檔複製 bytes，最後複製 `version.json`。

這個流程適合 build-time 同步，不適合 runtime reload：

- 新 World 可能在複製中途啟動。
- reload 可能遇到缺檔。
- 手動修改單一 bytes 時沒有版本邊界。
- generated Go code 與 bytes 可能不再成對。

### 2.3 現有 service 會捕獲啟動時配置

目前 item、effect、board handler 與 GM grant queue 都持有啟動時建立的物件。重讀檔案不會自動更新這些物件。

`variable.Service` 更包含 mutable dirty state、key locks 與 flush 生命週期。直接重建整個 service 可能遺失尚未 flush 的玩家變數。

因此不應直接在 `App` 上替換 service pointer。現有 setter 也沒有 reload 所需的同步保護。

### 2.4 Manifest 尚未保證 schema 相容

目前 [manifest.go](../../../pkg/dataconfig/manifest.go) 只記錄 config Git SHA、Luban version 與產出時間，沒有：

- bytes 檔案 checksum
- generated schema hash
- server binary 可接受的 schema 版本

Generated Go code 已編入 server binary。不同 schema 的 bytes 不應直接熱載入。

## 3. 目標與非目標

### 3.1 目標

- World 一次載入一個完整配置版本。
- 所有 domain loader 使用同一份 `GameConfigs`。
- 配置錯誤不覆蓋目前可用配置。
- 新請求使用新版本，進行中請求使用舊版本。
- 不遺失 variable dirty state。
- reload 不阻塞一般請求熱路徑。
- 版本與 schema 不相容時拒絕 reload。

### 3.2 非目標

- 不建立通用 reload framework。
- 不為單一實作新增大型 interface。
- 不在第一版支援所有 bytes 表熱更新。
- 不在沒有 migration 規則時修改既有玩家狀態。
- 不把 generated Go code 變成 runtime reload 資料。

## 4. 建議架構

### 4.1 原始 artifact loader

在 `pkg/dataconfig` 增加具體 loader：

```go
type Artifact struct {
    Manifest Manifest
    Tables   *generated.GameConfigs
}

func LoadDirectory(directory string) (*Artifact, error)
```

責任：

1. 讀取並驗證 `version.json`。
2. 一次呼叫 `generated.NewTables`。
3. 統一補上表名與路徑錯誤。
4. 驗證 checksum 與 schema hash。
5. 回傳完整 artifact。

`pkg/dataconfig` 只處理產物載入，不依賴 World domain。

現有 `LoadDefinitionsFromDirectory` 等函式可保留給工具與獨立測試，但應改成呼叫共用 loader。World 啟動路徑只能載入一次。

### 4.2 World Snapshot builder

新增 `cmd/world/internal/worldconfig`，負責把 raw tables 轉成 World 可用規則：

```go
type Snapshot struct {
    Version      string
    Variables    map[variable.ID]variable.Definition
    Effects      *effect.Definitions
    Items        *item.Definitions
    Board        board.Definition
    Selector     board.EventSelector
    ShopQuotes   map[string][]shop.Quote
    InitialMoney int64
}

func Build(artifact *dataconfig.Artifact) (*Snapshot, error)
```

`Build` 必須完成所有 domain 轉換後，再做交叉驗證：

- Gold variable 存在。
- Initial money 位於 Gold 的合法範圍。
- item action 指向已存在的 variable。
- shop quote 指向已存在的 item。
- 每個 shop space 都有 quote group。
- tech 引用的 effect 存在。
- board topology 完整閉環。
- 必要表存在。
- ID 不重複。

Snapshot 發布後不可修改。建構時複製 map、slice 與 action slice，避免底層 generated 資料被外部改寫。

### 4.3 Atomic Snapshot Store

使用 `sync/atomic.Pointer` 保存目前版本：

```go
type Store struct {
    current atomic.Pointer[Snapshot]
}
```

reload 流程在 atomic swap 前完成所有檔案 I/O、decode、domain 建構與驗證。發布只有一次 pointer swap。

每個 command 開始時取得一次 Snapshot，整個 command 固定使用該版本：

```text
request start
    ↓
snapshot := Current()
    ↓
整個 request 使用同一 snapshot
```

這保證不會在同一個 request 混用新舊 board、item、effect 規則。

## 5. Artifact 發布

### 5.1 版本化目錄

建議 runtime artifact 目錄：

```text
config/data/
├─ current.json
└─ releases/
   ├─ <config-sha>/
   │  ├─ version.json
   │  ├─ checksums.json
   │  ├─ Shared/
   │  └─ Board/
   └─ <next-sha>/
```

發布流程：

1. 寫入 `releases/<sha>.tmp`。
2. 複製完整 bytes、manifest、checksums。
3. 驗證所有必要檔案。
4. 將 `.tmp` rename 成完整 release 目錄。
5. 最後原子替換 `current.json`。

Server 只讀 `current.json` 指向的完整 release，不讀正在寫入的目錄。

### 5.2 Build-time 與 runtime 分離

現有 `sync-data-config.ps1` 同步 generated Go code 與 bytes，保留作為 build-time 工具。

另外建立 runtime publish 流程，runtime publish 只處理 bytes artifact：

- 不修改 `pkg/dataconfig/generated`。
- 不刪除目前正在使用的 release。
- 不直接覆蓋 `config/data/Shared/*.bytes`。

### 5.3 Schema hash

Manifest 增加：

```json
{
  "config_version": "...",
  "schema_hash": "...",
  "files": {
    "Shared/Item.bytes": "sha256:...",
    "Shared/Variable.bytes": "sha256:..."
  }
}
```

Server binary 嵌入 expected schema hash。reload 前必須確認：

```text
artifact.schema_hash == server.expected_schema_hash
```

不相容時拒絕 reload，要求部署新版 server binary。

## 5.4 Config 版本與 Client 相容性

Config 版本不等於 Client 版本。

例如：

```text
Client 1.0 / Protocol V1
Server 2.0 / Protocol V1 / Config B
```

如果 Config B 只修改商店價格、道具數值或事件機率，Client 不需要更新。

Config SHA 只回答：

```text
這批 bytes 來自哪個 PJA-Config commit？
```

Config SHA 不回答：

```text
Client 能不能解析 Server 封包？
Server binary 能不能讀取這批 bytes？
舊 Client 能不能理解新資料？
```

三種判定分開處理：

| 判定項目 | 負責版本 | 用途 |
|---|---|---|
| Client/Server 封包相容 | `protocol_version` | 判斷封包格式與訊息是否相容 |
| Server 能否讀取 bytes | `schema_hash` | 判斷 generated Go code 與 bytes schema 是否相容 |
| Client 能否理解新資料 | `client_contract` | 判斷 Client-visible enum、ID、資源與流程是否相容 |
| 目前使用哪批設定 | `config_version` | 追蹤、log、比較與 rollback |

Manifest 建議包含：

```json
{
  "config_version": "31c4a92b7e1f...",
  "schema_hash": "sha256:abc123...",
  "client_contract": 1,
  "compatibility": "backward-compatible"
}
```

判定例子：

```text
只修改 Server 數值：
config_version   A → B
schema_hash      X → X
client_contract  1 → 1
protocol         V1 → V1
結果             舊 Client 可繼續使用
```

```text
新增舊 Client 不認識的事件：
config_version   A → B
schema_hash      X → X
client_contract  1 → 2
結果             舊 Client 不可收到新事件，或必須更新
```

```text
修改欄位型別或 frame 格式：
protocol         V1 → V2
結果             必須走 Protocol 相容策略，可能要求更新 Client
```

Server 的判定概念：

```go
func validateClientCompatibility(manifest Manifest, protocolVersion uint16) error {
    if err := msg.ValidateProtocolVersion(protocolVersion); err != nil {
        return fmt.Errorf("validate protocol: %w", err)
    }
    if manifest.SchemaHash != expectedSchemaHash {
        return errors.New("config schema is not supported")
    }
    if manifest.Compatibility == "requires-client-update" {
        return errors.New("client update is required")
    }
    return nil
}
```

`compatibility` 不應只依賴人工填寫。PJA-Config CI 應檢查變更內容：

- 只修改 Server 數值欄位，保持 `client_contract` 不變。
- 新增 Client 不認識的 enum、message、道具資源或流程，要求提高 `client_contract`。
- 修改欄位型別、欄位順序或 frame 格式，必須提高 `protocol_version`。

Client 不需要知道 Server 使用哪個 Config SHA。Client 只需帶上自己使用的 Protocol version；Server 依 Protocol、schema 與 client contract 判斷是否允許使用。

## 5.5 遊戲執行期相容性與可重現性

配置 reload 不只改變讀檔內容，也可能改變遊戲結果。遊戲執行期必須記錄使用哪個配置版本，避免同一局遊戲、交易或 replay 無法重現。

### Session 固定配置版本

建議新遊戲或 match 建立時固定配置版本：

```text
玩家進入遊戲：Config A
遊戲進行中：Server reload Config B
該局完成：仍使用 Config A
下一局開始：使用 Config B
```

`match`、`game session` 或 board state 應保存：

```text
config_version
protocol_version
```

不應讓同一局遊戲因中途 reload 而混用新舊 board topology、tech 規則或其他狀態規則。

### Replay 與隨機結果

Replay 至少需要保存：

```json
{
  "config_version": "8aca768319bc...",
  "protocol_version": 1,
  "request_id": 12345,
  "random_result": 4,
  "player_id": 1001
}
```

只保存玩家輸入不夠。相同輸入在不同 Config 或不同隨機結果下，可能得到不同事件與獎勵。

正式遊戲可以繼續使用安全隨機；為了 replay，可保存已決定的 dice、event id、drop result 等結果。Replay 直接使用已記錄結果，不必重建完整隨機數流。

### 重要資料保存配置版本

以下資料建議保存 `config_version`：

- match 或 game session。
- shop interaction。
- combat result。
- reward record。
- GM grant。
- replay record。
- audit record。

例如 shop interaction 建立時使用 Config A，之後即使 reload 到 Config B，玩家確認購買仍使用建立交易時的價格。現有交易已保存價格，補上配置版本後可追蹤該價格來源。

### 配置生效時間

不是所有配置都適合立即生效。Artifact 可增加生效時間：

```json
{
  "config_version": "31c4a92b7e1f...",
  "effective_at": "2026-09-16T12:00:00Z"
}
```

建議分類：

- 商店價格、活動開關：可立即生效。
- board topology、研究需求、玩家資源上下限：新局或重啟生效。
- 賽季、限時活動：依 `effective_at` 生效。

### 多 World 灰度

多台 World 可能短時間使用不同配置：

```text
World 1：Config A
World 2：Config B
World 3：Config A
```

需要明確選擇策略：

- 全服同時切換。
- 按 World 灰度。
- 按玩家固定分流。

按玩家分流時，玩家必須固定使用同一版本，不能每次 request 重新隨機分配。跨 World 移動時也要遵守原本的分流規則。

每台 World 都必須自行載入、驗證與回報結果，不能只相信發布服務已成功。不同 World 的 server binary、schema 或資料掛載可能不同。

### 舊資料與 ID 保護

新配置不能讓舊資料引用失效：

```text
DB / Redis / queue 仍引用 Item ID 1001
新 Config 刪除 Item ID 1001
玩家重登或 GM queue 執行失敗
```

發布前應檢查仍被使用的：

- Item ID。
- Effect ID。
- Variable ID。
- Space ID。
- Tech ID。
- Event ID。

第一版禁止刪除仍被引用的 ID。需要刪除時，先標記 deprecated，再做 migration 或確認所有引用已清除。

### 經濟數值保護

涉及貨幣、價格、掉落與獎勵的配置，除了型別驗證，也應檢查變更幅度：

- 價格不得小於 0。
- 貨幣獎勵不得超過上限。
- cooldown 不得為負數。
- 掉落率總和必須符合規則。
- shop item 必須存在。
- reward 不得引用已刪除內容。
- 超過設定變更幅度時要求人工確認。

例如價格從 `100` 變成 `1000000`，格式合法但可能是企劃誤操作。這類變更應在 dry-run 或發布審核階段被標出。

### Client 資源版本

Server config version 與 Client asset version 分開管理。

```text
server_config_version
client_asset_version
protocol_version
client_contract
```

Server Config 新增道具 ID 時，舊 Client 可能沒有對應圖片、名稱或 UI。Protocol 仍相容，不代表畫面一定能正常顯示。

若需要限制，可在登入或內容查詢回傳：

```json
{
  "required_client_contract": 2,
  "required_asset_version": "2026.09"
}
```

舊 Client 不符合時，可以限制新活動，也可以要求更新；不應讓舊 Client 收到無法理解的資料。

## 6. Reload 執行流程

```text
current.json 變更
    ↓ debounce / 合併重複事件
檢查是否已有 reload
    ↓
讀取新 release
    ↓
驗證 manifest、schema、checksum
    ↓
建立 candidate Snapshot
    ↓
執行跨表驗證與 old/new transition policy
    ↓
atomic publish
    ↓
記錄版本、延遲、結果
```

規則：

- 同一時間最多一個 reload。
- 相同版本直接忽略。
- fsnotify 重複事件合併。
- reload 失敗保留舊 Snapshot。
- 磁碟 I/O 不放在 mutex 內。
- watcher goroutine 綁定 `baseApp.Context()`，shutdown 時必須退出。
- 手動 reload 與檔案 watcher 共用同一個 `Reload()` 入口。
- 第一版不自動無限 retry；下一次 marker 更新或人工觸發再試。

現有 `pkg/configwatch` 綁定 Viper 單一 YAML 檔，不建議改造成 bytes watcher。新增小型 data-config watcher 即可，底層仍使用現有 fsnotify 依賴。

## 7. Reload 白名單

第一版採白名單。未列入的配置變更一律拒絕 reload。

| 表 | 第一版策略 | 原因 |
|---|---|---|
| `Shared/Shop` | 允許 | 只影響新建立的 shop interaction；既有 interaction 已保存價格 |
| `Board/Event` | 允許 | 只影響下一次事件選擇 |
| `Shared/Effect` | 允許更新，禁止刪除 ID | persisted effect 可能仍引用舊 ID |
| `Shared/Item` | 允許更新，禁止刪除 ID | GM queue 與 inventory 可能仍引用舊 ID |
| `Shared/GameRules` | 欄位級判定 | InitialMoney 與既有進度規則風險不同 |
| `Shared/Variable` | 不允許 | default、min、max 會改變既有玩家資料語意 |
| `Board/BoardTopology` | 不允許 | persisted `SpaceID` 可能失效 |
| `Board/Tech` | 不允許 | 玩家已有 research state |
| `Board/RoadStar` | 不允許 | 涉及 persisted board state |
| `Board/Building` | 不允許 | 涉及 persisted board state |
| `LocalizationText` | 暫不處理 | 目前 World 流程未使用 |

需要支援 Variable、Topology、Tech reload 時，先定義資料 migration 與 old/new transition validator。

## 8. Stateful service 處理

### 8.1 Variable service

`variable.Service` 保留單一實例，不因 reload 重建。它的 dirty map、key lock、flush goroutine 必須持續存在。reload 時先 flush dirty values，再以寫鎖原子替換 definitions、目前版本與 migration chain；一般 Variable 操作持有讀鎖，因此不會混用新舊規則。

Variable reload 至少需要：

- ID 不得刪除。
- default 變更需明確規則。
- min/max 收窄前掃描既有玩家值。
- dirty values 必須重新驗證。
- reload 與 flush 必須有清楚的鎖定順序。

### 8.2 Item、Effect、Board

這些規則物件可在 Snapshot 中建立新版本，但 handler 不能永遠捕獲初始物件。

以下入口都要在操作開始時取得目前 Snapshot：

- Gate board payload handler。
- GM grant item queue。
- 其他直接使用 item/effect/board rules 的背景工作。

## 9. 分階段實作

### Phase 1：統一載入，行為不變

狀態：已實作。World 在啟動時只讀取一次完整 bytes tables，Variable、Effect、Item、Board、Shop 與 InitialMoney 都由這同一份 tables 建立；原有以目錄載入的公開函式仍保留，供其他工具使用。

修改範圍：

- `pkg/dataconfig`：新增 `LoadDirectory` 與 `Artifact`。
- `cmd/world/internal/worldconfig`：新增 Snapshot builder。
- `internal/game/itemconfig`：保留從 tables 轉換的入口。
- `internal/game/effectconfig`：使用既有從 tables 轉換的入口。
- `internal/game/boardconfig`：拆分 raw tables 載入與 definition conversion。
- `cmd/world/internal/variable`：新增從 tables 建立 definitions 的入口。
- `cmd/world/internal/services/services.go`：只載入一次 artifact。

不做：

- 不啟用 watcher。
- 不替換執行中 service。
- 不修改 generated Go code。

驗收：

- 每個 bytes 表只讀一次。
- World 啟動成功與既有結果一致。
- 任一表失敗，整個 Snapshot 建立失敗。
- board、shop、initial money 使用同一份 tables。

### Phase 2：安全 artifact 發布

狀態：已實作。使用 `go run ./tools/configartifact -source config/data -root config/data` 發布；工具會先驗證完整 tables，再建立 `releases/<config-sha>/`、寫入 `checksums.json`，最後才原子更新 `current.json`。`schema_hash` 由編譯進 binary 的 `generated/GameConfigs.go` 內容計算，生成 code 改變時舊 artifact 會被拒絕。

World 預設要求 `current.json`，避免執行期直接讀取正在同步的 raw bytes。只有配置明確設定 `data_config.allow_legacy_directory: true` 時才保留 legacy 目錄讀取，僅供本機開發與 Docker 範例使用。

若已有 current artifact，發布器也會在更新 `current.json` 前執行 reload transition validator；不相容版本不會改變目前指標。

PowerShell 部署入口：`powershell -ExecutionPolicy Bypass -File .\scripts\publish-data-config.ps1`。

- 建立 versioned release directory。
- 增加 checksums 與 schema hash。
- 拆分 build-time sync 與 runtime publish。
- `current.json` 最後原子更新。

驗收：

- publish 中途失敗不影響目前 release。
- 啟動與 reload 不會讀到半套資料。
- schema 不相容時明確拒絕。

### Phase 3：安全子集 reload

狀態：已實作。Snapshot Store 會原子切換已驗證 artifact；`current.json` watcher 以 debounce 合併事件，僅允許 Shop、Event、Effect、Item 變更。Board request 與 GM grant queue 都在命令開始時固定取得已完成 item、effect、board 與 shop domain 轉換的 World Snapshot，不會在請求中重建規則；成功、失敗、拒絕、耗時與最後成功時間均有 metrics，成功切換也會記錄新舊版本與 schema hash。

World 會在建立與發布 Snapshot 前，先轉換 variable、item、effect、board 與 shop domain 規則，並驗證 Gold 與 InitialMoney 範圍、item action 的 variable、shop quote 的 item 與 shop space 群組、以及 technology 的 effect。任一驗證失敗時，candidate 不會發布。

Item、Effect 與 Event 的 ID 不可在 reload 時刪除；若需要移除，應先完成持久化資料與佇列的 migration。

Shop reload 會驗證每筆商品都仍引用存在的 Item ID。

### Topology migration

棋盤的 `SpaceID` 是玩家棋盤狀態、replay 與 audit record 的持久化契約。調整名稱、圖示或位置不需要 migration；刪除或改名 `SpaceID` 時，必須由**新 artifact** 提供搬遷資料。`research_tech_id` 也是持久化契約；移除或改名 Tech ID 時，在同一份 `migrations/topology.json` 的 `tech_ids` 以舊 ID 對應新 ID。搬遷會原子更新棋盤狀態、研究 effect 的 source ID 與 `player_config_version(scope=board)`。RoadStar 只持久化 Space ID，已由 `space_ids` 覆蓋；Building 目前沒有玩家持久化欄位。

```text
config/data/releases/<new-config-sha>/
├─ Board/BoardTopology.bytes
└─ migrations/topology.json
```

`topology.json` 記錄舊版本的格子要搬到新版本哪個格子：

```json
{
  "from_config_version": "old-config-sha",
  "space_ids": {
    "factory_old": "factory_main",
    "road_07": "road_06"
  }
}
```

artifact 發布前要驗證：被移除的舊 `SpaceID` 都有 mapping、mapping 目標存在於新 topology、且不允許目標指向另一個已移除 ID。缺少 mapping 時拒絕發布，不能靜默將玩家送回起點。

Topology migration 不是熱重載。先停止所有 World，再以 `go run ./tools/configartifact -topology-migration`（或 `publish-data-config.ps1 -TopologyMigration`）發布；啟動前以 `go run ./tools/configartifact -verify-topology-lineage`（或 `publish-data-config.ps1 -VerifyTopologyLineage`）確認 artifact 歷史與 migration steps 完整，最後才啟動 World。發布模式只額外允許 `Board/BoardTopology.bytes` 變更，且必須包含 `migrations/topology.json`，其他不安全變更仍會被拒絕。

後續 topology migration（已存在 topology artifact chain）改用可搬遷性 gate：

```powershell
.\scripts\publish-data-config.ps1 -TopologyMigration -RequireTopologyMigratable
```

它以目前 artifact 與 DB 版本分布做唯讀 preflight；若出現無法走到目前版本的玩家資料，候選 artifact 不會發布。

在評估歷史 artifact 是否仍必須保留時，可查看玩家盤面版本與目前 lineage 相依：

```powershell
.\scripts\initialize-board-topology.ps1 -Status -RetentionReport
```

每個版本會列出 checkpoint 數與 `in_lineage`，報表再列出每個 artifact 的保留原因。現行 chained lineage 下，目前 artifact 仍直接相依所有父 artifact，因此報表的 `prunable_artifacts` 會是 0；要真的清理歷史 release，必須另做 lineage compaction，不能直接刪目錄。

lineage compaction 會發布新的 Config SHA root artifact：`.\scripts\publish-data-config.ps1 -CompactTopologyLineage`。它不帶 parent，腳本會先確認所有玩家已在目前版本；舊 release 仍由後續明確清理流程處理。

完成 compaction 後，以 `.\scripts\cleanup-data-config-releases.ps1` 列出可清理 release；確認清單後才加 `-Apply` 刪除。`-Apply` 會先以 `WORLD_MYSQL_DSN` 驗證所有玩家已在目前版本。

若要主動縮短保留期，可在 World 全部停止時執行逐段離線搬遷：

```powershell
.\scripts\initialize-board-topology.ps1 -MigrateCurrent
```

它會先掃描所有盤面版本，確認每一種版本都能走到目前 artifact；preflight 通過後才重用 artifact lineage 的每一段 migration，逐筆交易式更新盤面。遇到 legacy 或未知版本時不會開始寫入，修正後可安全重跑。

離線搬遷預設不設全程 timeout，避免大量玩家搬到一半逾時；`0` 表示不設期限，負值會被拒絕。若部署窗口需要上限，可傳入 `.\scripts\initialize-board-topology.ps1 -MigrateCurrent -Timeout 30m`。

工具會先輸出 `preflight_board_states`，之後每完成一批輸出 `processed_board_states`、`migrated_board_states`、`migrated_topology_steps` 與 `last_player_id`，可用於監控長時間作業進度。

所有批次完成後，工具會再讀取版本分布並輸出 `postflight_board_states`；只要仍有非目前版本的盤面，整個指令會失敗。

可用 Ctrl+C 或服務終止訊號取消離線作業；每位玩家的搬遷仍是獨立交易，取消後重新執行會從目前已保存的版本安全續跑。

若要略過已確認完成的前段，可將最後輸出的 ID 帶入 `-StartAfterPlayerID <id>`；此參數只可搭配 `-MigrateCurrent`，且每次仍會先做全表版本 preflight。

停機前可先執行相同的唯讀 preflight：

```powershell
.\scripts\initialize-board-topology.ps1 -Status -RequireMigratable
```

任何盤面版本無法走到目前 artifact 時會以非零結束碼失敗，不會寫入資料。

搬遷後可用嚴格 gate 確認所有盤面已在目前版本，這是考慮清理舊 artifact 前的必要條件：

```powershell
.\scripts\initialize-board-topology.ps1 -Status -RequireCurrent
```

只要仍有歷史版本或未知 checkpoint，指令就會以非零結束碼失敗；它不會自行刪除 artifact。

#### 歷史 migration 鏈

如果玩家很久未登入，可能仍停在 A 版本，但目前已發布 C：

```text
A: factory_old
   ↓ A → B
B: factory_main
   ↓ B → C
C: factory_center
```

B artifact 保存 `factory_old → factory_main`；C artifact 保存 `factory_main → factory_center`。沒有變動 SpaceID 的版本可視為 identity migration，不一定需要檔案，但系統仍需能把玩家的 `player_config_version(scope=board)` 推進到目前版本。

實作可選兩種策略：

- 逐段執行 A→B→C：每個 artifact 保存前一版 mapping。
- 目標版展開：C 保存所有仍可能存在的舊版本到 C 的 mapping。

第一版採逐段策略，artifact 只需攜帶直接前身的 mapping，規則最小。artifact 發布時會記錄父版本，World 從目前 artifact 回溯 immutable release，對久未登入玩家逐段套用 A→B→C；沒有 SpaceID 變更的版本會形成 identity step。父 release 不可在仍有玩家使用其版本時刪除。找不到玩家版本的下一步時必須拒絕讀取，不能直接將其標記為目前版本。玩家每成功一段搬遷，都在同一筆交易更新其 `SpaceID` 與 `player_config_version(scope=board)`；寫入失敗不得更新該段版本。

先支援 Shop、Event、Effect 更新、Item 更新。

加入：

- Snapshot Store。
- watcher 與 debounce。
- reload mutex 或 singleflight。
- old/new transition validator。
- reload metrics 與 structured log。
- board handler、GM queue 的 current Snapshot 讀取。

驗收：

- 新 request 使用新版本。
- 進行中 request 使用舊版本。
- invalid artifact 不覆蓋舊版本。
- concurrent reload 不產生 data race。
- GM grant queue 使用新 item rules。

### Phase 4：擴大範圍

狀態：多 World 定時發布已實作。PJA-Config 建置時可指定 `-EffectiveAt '2026-09-16T12:00:00+08:00'` 寫入 manifest。artifact 會先成為 `current.json` 指向的候選版本；World 會在本機完整載入、驗證與預建候選快照，但在該 UTC 時刻前仍使用父版本。時間到後，各 World 以既有請求寫鎖完成 Variable reload 與原子切換；較新的 `current.json` 會取消尚未生效的候選計時器。首次發布不可設定未來 `effective_at`，因為沒有父 release 可繼續服務。

狀態：Variable migration 與熱重載已實作。變更 `Shared/Variable.bytes` 必須使用專用發布模式，artifact 需提供從父版本到目前版本的 `migrations/variables.json`；目前僅支援明確列出的 bounds `clamp` 規則，ID、default 與 client sync 設定仍不允許變更。World 每次存取玩家 Variable 前，會以 `player_config_version(scope=variables)` 讀取 checkpoint，並在同一筆交易依歷史鏈逐步 clamp、更新 checkpoint；已有 sparse variable 但缺 checkpoint 時會拒絕猜測版本，沒有 sparse variable 的新玩家則直接建立目前版本。watcher 僅在完整 migration 驗證通過後才會 flush dirty values 並切換 Variable definitions。需要在 artifact compaction 前主動完成全量搬遷時，先執行 `.\scripts\migrate-player-variables.ps1 -Status -RequireMigratable` 預檢，再於停止 World 後執行 `.\scripts\migrate-player-variables.ps1 -MigrateCurrent`；工具以 player ID 批次處理，可用 `-StartAfterPlayerID` 續跑，且會拒絕未有 checkpoint 的 sparse variable 資料。

尚未實作的擴充範圍如下；它們都需要先定義資料相容規則與 migration：

- Variable default 變更遷移；bounds `clamp` 已支援。
- Board topology 線上熱重載；目前仍要求停止 World 後發布。
- Tech progress 線上熱重載。
- Item／Effect ID 移除。
- 通用 persisted state migration。

## 10. 測試與觀測

### 10.1 測試

最低測試集合：

- loader 每個檔案只讀一次。
- 任一 bytes 缺少時 LoadDirectory 失敗。
- 跨表引用錯誤時 Build 失敗。
- invalid candidate 不覆蓋舊 Snapshot。
- `current.json` watcher 收到有效 artifact 時發布新的 World Snapshot；無效 marker 時保留舊 Snapshot。
- 相同 version 不重載。
- concurrent reload 只執行一次。
- request 固定使用同一 Snapshot。
- 未到 `effective_at` 的候選會先驗證但不提前切換；時間到才發布。
- 新候選會覆蓋尚未生效的定時發布。
- schema mismatch 被拒絕。
- Item/Effect ID removal 被拒絕。
- watcher shutdown 不洩漏 goroutine。

依專案規範執行：

```powershell
go test -race -shuffle=on -timeout=5m ./...
```

### 10.2 Metrics

```text
data_config_reload_total{result="success|failed|rejected"}
data_config_reload_duration_seconds
data_config_last_reload_timestamp_seconds
```

不要把完整 config SHA 放入 Prometheus label，避免 label cardinality 持續增加。版本寫入 log、debug endpoint 或 readiness detail。

### 10.3 Log

每次 reload 記錄：

```text
old_version
new_version
schema_hash
duration
result
error
```

Reload 失敗時服務繼續使用舊 Snapshot，不應因舊 Snapshot 有效而標記為 unready。首次啟動載入失敗則 World 啟動失敗。

## 11. 驗收條件

方案完成後，必須符合：

1. World 啟動只建立一份完整 `GameConfigs`。
2. Board definition、shop quotes、initial money 來自同一版本。
3. Server 不直接讀取正在同步中的 bytes 目錄。
4. Invalid reload 不覆蓋目前有效配置。
5. 新舊 request 不混用 Snapshot。
6. Variable dirty state 不因 reload 遺失。
7. Generated Go schema 與 bytes schema 不相容時拒絕 reload。
8. watcher 可正常停止，沒有 goroutine leak。
9. reload 結果可透過 metrics 與 log 排查。

## 12. 最終建議

目前已完成統一載入、安全 artifact 發布、白名單 reload、Variable／topology migration 與多 World `effective_at` 定時發布。

不要直接做「所有配置都可熱更新」。後續仍應依下列順序擴大範圍：

```text
統一載入
    ↓
原子發布 artifact
    ↓
建立 immutable Snapshot
    ↓
白名單 reload
    ↓
資料 migration 後擴大範圍
```

## 13. 指定版本切換與備份

指定既有 release 切換時，不可由腳本直接覆寫 `current.json`。必須經過 Server 的 `SnapshotStore.SwitchTo`，驗證版本、manifest、checksum、schema 與 reload 相容性後，才會原子更新 marker 並發布 snapshot：

```powershell
.\scripts\backup-data-config-release.ps1 -Root config\data -Version <config-sha>
.\scripts\rollback-data-config-release.ps1 -Root config\data -Version <config-sha>
```

`backup-data-config-release.ps1` 預設只建立備份，不修改目前配置。Rollback 只切換已存在且通過驗證的 artifact，不自動搬遷或還原玩家資料；不相容版本會拒絕切換。
