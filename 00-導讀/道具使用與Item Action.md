# 道具使用與 Item Action

現行道具使用行為；完整方案背景見[方案草稿](../90-參考附件/方案草稿/道具取得與使用系統規範.md)。

- 流程：驗證玩家與道具 → 扣除背包數量 → 執行 Item Action → 回傳變更集。
- `AdjustVariable`：`Param1` 為 Variable ID，`Param2` 為單個道具的絕對變更量。
- 變更集可同時回傳 inventory 與 client-visible variable 更新。
- Item Action 不等同 Effect；其他 Item Type、`Param3` 與 `ItemSource` 尚未支援。

主要實作位於 `cmd/world/internal/itemaction` 與 `cmd/world/internal/board/use_item.go`。
