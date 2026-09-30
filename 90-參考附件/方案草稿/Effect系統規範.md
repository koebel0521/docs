# Effect 系統現行規範

## 0. 目前進度

已完成 `internal/game/effect` 的數值聚合核心、`effectconfig` Luban loader，以及 World 的持久化、來源替換、清除與讀取 Service。科技完成時會建立科技來源，並在擲骰時套用 `FactoryProductionSteps`。讀取玩家 Effect 時會在同一個 transaction lazy 清除過期資料；只有累積量真的造成儲存壓力時，才補背景批次清除。

## 1. 目的與邊界

Effect 是「多個來源共同影響某個遊戲數值或規則」的聚合系統。它保存每個來源的貢獻，並在需要時依 `EffectId` 算出有效結果。

Effect 不做以下事情：

- 不直接寫入或修改 Variable；Variable 是玩家持有的基礎值，Effect 是計算中的修正值。
- 不處理 TCP Protocol、封包或 Client 顯示。
- 不由 Item 直接變更。Item 使用後只能路由到其明確對應的業務系統；目前只支援 `AdjustVariable`。

因此，消費金幣時讀取的是 Variable；計算工廠產能、科技速度或商店規則時，業務系統傳入基礎值後向 Effect 查詢有效值。

```text
Variable / 業務基礎值
        │
        ├── Effect.Resolve(EffectId, base)
        ▼
有效遊戲數值或規則
```

## 2. 核心模型

```go
type SourceType uint16

type Source struct {
	Type SourceType
	Id   uint64
}

type Contribution struct {
	EffectId  effect.ID
	Value     int64
	ExpiresAt *time.Time
}

type Instance struct {
	Id        uint64
	OwnerId   uint64
	Source    Source
	EffectId  effect.ID
	Value     int64
	ExpiresAt *time.Time
}
```

- `EffectId`：Effect 定義識別碼，表示它影響哪一種數值或規則。
- `OwnerId`：目前固定為玩家 ID；日後若有公會或世界效果，再擴充 Owner 類型，不提前抽象。
- `SourceType`：來源分類，例如科技、建築、活動、暫時 Buff。
- `Source.Id`：該玩家範圍內來源的穩定識別碼。建築必須使用建築實例 ID，不可使用建築設定 ID。
- `Instance.Id`：持久化後的單筆 Effect ID，僅供精確移除與除錯。

同一個 `(OwnerId, SourceType, Source.Id, EffectId)` 最多一筆。若一個來源需要改變效果，直接替換來源的完整貢獻集合；不要累加一堆難以回收的臨時資料。

## 3. Service 操作

Effect runtime 提供以下操作：

| 操作 | 用途 |
| --- | --- |
| `ReplaceSource` | 原子地以一組 Contributions 覆蓋來源現有的全部 Effect；科技升級、建築變更與 Buff 刷新都走此操作。 |
| `RemoveSource` | 依 `(SourceType, SourceId)` 清除指定來源。 |
| `ClearSourceType` | 清除某玩家某一來源類型，例如活動結束時移除所有活動效果。 |
| `RemoveInstances` | 依 `InstanceId` 精確移除，限同一 Owner。 |
| `Snapshot` | 取得尚未過期的 Effect 實例，供業務系統或工具檢視。 |
| `Resolve` | 對指定 `EffectId` 將 Contributions 聚合到呼叫端提供的基礎值。 |

不提供裸露的 `Add` 作為主要業務 API。一次性的 Buff 應建立一個新的 `Source` 後呼叫 `ReplaceSource`；這可自然處理重送、刷新與來源清除。

## 4. 聚合規則

目前 Luban `Shared/Effect` 已定義：

- `Operator`: `Plus` 或 `Minus`
- `Quantifier`: `Point` 或 `Percent`
- `Color`: 顯示或分類資料，Effect 核心不解讀它

MVP 僅支援加減型數值 Effect，固定公式如下：

```text
pointDelta   = Σ signed(Point contributions)
percentDelta = Σ signed(Percent contributions)
result       = (base + pointDelta) * (100 + percentDelta) / 100
```

整數除法採 Go 的截斷規則。Effect 核心不自行 clamp；由擁有該數值的業務系統在結果後套用自己的上下限。

設定中的 `Operator = Null` 不是數值聚合規則。現有 `ShopGroup` 即屬此類，MVP 不得把它當成 `Plus`、`Minus` 或任意覆蓋值。要啟用這類規則前，先在 PJA-Config 增加明確的 `StackMode` 與 `Priority`，再實作對應的 selector resolver。

## 5. 保存與失效

World 的持久化資料表為 `player_effect`：

```text
instance_id        bigint primary key
player_id          bigint not null
source_type        smallint not null
source_id          bigint not null
effect_id          int not null
value              bigint not null
expires_at         datetime null
updated_at         datetime not null
unique(player_id, source_type, source_id, effect_id)
index(player_id, effect_id, expires_at)
```

- `ReplaceSource`、`RemoveSource`、`ClearSourceType` 與 `RemoveInstances` 必須直接在資料庫交易完成。
- `Resolve` 與 `Snapshot` 載入資料時，會在同一個 transaction lazy 清除該玩家的過期列；累積量真的造成儲存壓力時，才新增背景批次清除，且不能影響正確性。
- 每次來源變動後使該玩家的 Effect 快取失效。MVP 可不做跨請求快取，先以正確性為準。
- 本次 deploy 只透過既有 AutoMigrate 新增 `player_effect`，不修改舊表或舊資料；rollback 到舊版時保留這張未使用的表即可。

## 6. 程式位置與相依方向

```text
internal/game/effect/
├─ model.go          Source、Contribution、Instance、Definition
└─ resolve.go        純聚合、過期判斷與設定驗證

internal/game/effectconfig/
└─ load.go           將 Luban EffectConfigTable 轉成 effect.Definition

cmd/world/internal/effect/
├─ service.go         Owner 驗證、Store 協調與快取
└─ store.go           World 所需的持久化介面

internal/game/effect/*_test.go
└─ 純 Resolve 與設定轉換測試
```

`internal/game/effect` 不得 import `cmd/world/internal`、資料庫、Protocol 或 `Variable`。World Handler 負責把業務事件轉成 `ReplaceSource` 等命令；工具只能使用 `internal/game/effect` 與 `effectconfig`。

## 7. 實作順序

1. 建立 `internal/game/effect`：型別、Effect 定義驗證、Point／Percent 的純聚合測試。
2. 建立 `effectconfig` loader，拒絕未知 `EffectId` 與 MVP 不支援的 `Operator = Null` 聚合呼叫。
3. 在 World 建立 `player_effect` Store、`ReplaceSource`／移除操作與過期查詢測試。
4. 先接一個既有來源（建議科技），在其變更時以 `ReplaceSource` 更新；讀取科技速度時接 `Resolve`。
5. 目前以 `internal/game/effect` 與 `internal/game/effectconfig` 的測試驗證聚合結果；尚未建立獨立 simulator。
6. 確認 ShopGroup 的覆蓋與優先權企劃規則後，再擴充非數值 Effect。

## 8. 與 Item Action 的關係

Item Action 仍是路由機制，不是 Effect 的別名。若未來企劃允許某類道具建立暫時 Buff，才新增明確的 Item Action Type 與其 World Handler；Handler 會建立一個受控的 Buff Source 後呼叫 Effect Service。這是新的需求，不能由現在的 `AdjustVariable` 推論或混用。
