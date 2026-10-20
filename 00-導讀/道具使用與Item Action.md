# 道具使用與 Item Action

現行道具使用行為；完整方案背景見[方案草稿](../90-參考附件/方案草稿/道具取得與使用系統規範.md)。

- 流程：驗證玩家與道具 → 扣除背包數量 → 執行 Item Action → 回傳變更集。
- `AdjustVariable`：`Param1` 為 Variable ID，`Param2` 為單個道具的變更量，必須為正數（載入時 `Amount <= 0` 會被拒絕），所以只能增加不能減少。
- 變更集可同時回傳 inventory 與 client-visible variable 更新。
- Item Action 不等同 Effect；其他 Item Type、`Param3` 與 `ItemSource` 尚未支援。

動作執行器（`item.Executor`、`item.Handler`）在 `internal/gamecore/item`；World 的 AdjustVariable 轉接器在 `cmd/world/internal/itemaction`，使用流程在 `cmd/world/internal/board/use_item.go`。Handler 由組合根（`services.Start`）註冊。
