---
tags: [project, configuration, deployment, rollback]
created: 2026-09-16
---

# 配置 Artifact 發布與回退

本文件涵蓋 `config/data` 版本化 artifact 的發布、拓樸搬遷、備份、回退與清理。Server 只使用 `current.json` 指向的完整 release；不要直接修改 marker 或正在同步中的 bytes。

World 預設要求 `current.json`。只有本機開發或 Docker 範例配置明確設定 `data_config.allow_legacy_directory: true` 時，才允許直讀 raw bytes；production 配置不得啟用此相容模式。

## 1. 一般發布

```powershell
.\scripts\publish-data-config.ps1 -Source config\data -Root config\data
```

發布流程會先驗證所有 bytes、checksum、schema 與跨表規則，再建立不可變 release，最後原子更新 `current.json`。`sync-data-config.ps1` 只適合 build-time 同步，不是 runtime 發布工具。

可用 `-VerifyTopologyLineage` 檢查目前 artifact 的拓樸 lineage：

```powershell
.\scripts\publish-data-config.ps1 -Root config\data -VerifyTopologyLineage
```

## 2. 拓樸變更

拓樸變更必須先停止所有 World，並提供完整 `migrations/topology.json`。發布前可檢查所有既有盤面是否可走到目前 artifact：

```powershell
.\scripts\publish-data-config.ps1 `
  -Root config\data `
  -TopologyMigration `
  -RequireTopologyMigratable `
  -TopologyBaselineDSN $env:WORLD_MYSQL_DSN
```

發布後重新啟動 World。盤面版本初始化與離線搬遷：

```powershell
.\scripts\initialize-board-topology.ps1 -Root config\data -Status
.\scripts\initialize-board-topology.ps1 -Root config\data -Apply
.\scripts\initialize-board-topology.ps1 -Root config\data -MigrateCurrent -Timeout 30m
```

`-MigrateCurrent` 可用 `-StartAfterPlayerID` 續跑；每筆盤面搬遷是交易式且可安全重試。完成後用 `-Status -RequireCurrent` 驗證所有盤面已到目前版本。

## 3. lineage 壓縮

當所有盤面都已在目前版本，才可建立無 parent 的新 root artifact：

```powershell
.\scripts\publish-data-config.ps1 `
  -Root config\data `
  -CompactTopologyLineage `
  -TopologyBaselineDSN $env:WORLD_MYSQL_DSN
```

壓縮只切斷配置 lineage，不搬遷玩家資料，也不會自動刪除舊 release。

## 4. 備份與回退

先備份目前 marker 與指定 release：

```powershell
.\scripts\backup-data-config-release.ps1 -Root config\data -Version <config-sha>
```

回退必須透過 Go 驗證流程，不可直接覆寫 `current.json`：

```powershell
.\scripts\rollback-data-config-release.ps1 -Root config\data -Version <config-sha>
```

切換前會檢查 release 版本、manifest、checksum、schema 與 reload 相容性；失敗時保留目前 snapshot。回退只切換配置，不自動還原玩家資料。拓樸或資料格式不相容時，應先完成對應 migration。

## 5. 舊 release 清理

先產生只讀保留報表：

```powershell
.\scripts\initialize-board-topology.ps1 `
  -Root config\data `
  -Status `
  -RetentionReport
```

確認盤面已全部搬到目前版本後，再執行 cleanup 的 dry run：

```powershell
.\scripts\cleanup-data-config-releases.ps1 -Root config\data
```

確認候選清單、備份 `current.json` 與清單後，才加 `-Apply`。`-Apply` 會再次檢查玩家盤面版本；不符合時拒絕刪除。

清理是破壞性操作。保留中的 current lineage、仍被玩家引用的版本、未完成搬遷或無法辨識的 legacy 版本不可清理。

## 6. 常見失敗判讀

- `schema is not supported`：Server binary 與 bytes 需要同批更新，先部署相容 binary。
- `config file ... cannot be reloaded`：變更超出白名單，改走重啟或資料 migration。
- `topology migration ...`：缺少 mapping、mapping 目標不存在，或玩家版本無法到達目前版本。
- `Player topology version verification failed`：先初始化 baseline 或完成離線搬遷。
- cleanup 拒絕：不要手動刪目錄，先處理仍在 lineage 或仍被資料引用的版本。
