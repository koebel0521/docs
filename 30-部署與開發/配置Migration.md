# 配置 Migration

現行配置版本遷移行為；完整相容性設計見[方案草稿](../90-參考附件/方案草稿/bytes配置統一與熱重載方案.md)。

- Topology migration 只處理棋盤拓撲版本，需停止所有 World 後執行。
- Variable migration 依 checkpoint 與 migration chain 套用允許的 bounds clamp。
- 未知 checkpoint、缺少 migration 或不安全欄位變更時拒絕執行，不猜測版本。
- 執行前先用 `scripts/migrate-player-variables.ps1 -Status -RequireMigratable` 預檢。

發布、驗證與回退入口見[配置操作入口](配置操作入口.md)。
