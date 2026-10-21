---
tags: [project, configuration, deployment, rollback]
created: 2026-09-16
---

# 配置 Artifact 發布與回退

本文件涵蓋 `config/data` 版本化 artifact 的發布、拓樸搬遷、備份、回退與清理。Server 只使用 `current.json` 指向的完整 release；不要直接修改 marker 或正在同步中的 bytes。

所有設定維運腳本的相對 `-Root`／`-BackupRoot` 都以倉庫根目錄為基準（不是目前所在目錄）；部署主機上請傳絕對路徑。

World 預設要求 `current.json`。只有本機開發或 Docker 範例配置明確設定 `data_config.allow_legacy_directory: true` 時，才允許直讀 raw bytes；production 配置不得啟用此相容模式。

## 1. 一般發布

```powershell
.\scripts\publish-data-config.ps1 -Source config\data -Root config\data
```

發布流程會先驗證所有 bytes、checksum、schema 與跨表規則，再建立不可變 release，最後原子更新 `current.json`。`sync-data-config.ps1` 只適合 build-time 同步，不是 runtime 發布工具。

實際發布前先加 `-DryRun`（對應 `configartifact -dry-run`）：只做來源驗證與換版規則檢查，不寫 releases、不動 `current.json`，輸出多一行 `dry_run=true`：

```powershell
.\scripts\publish-data-config.ps1 -Source config\data -Root config\data -DryRun
```

`-dry-run` 只適用發布類模式，不可與 `-switch-version`、`-verify-topology-lineage` 併用（會回用法錯誤，結束碼 2）。

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

發布後重新啟動 World。盤面版本檢查與初始化／離線搬遷：

```powershell
.\scripts\initialize-board-topology.ps1 -Root config\data -Status
.\scripts\initialize-board-topology.ps1 -Root config\data -MigrateCurrent -Timeout 30m
```

`initialize-board-topology.ps1` 沒有 `-Apply`，也不是不帶旗標就初始化：`-Status`（只報告各拓樸版本的盤面數）與 `-MigrateCurrent`（把持久化盤面搬到目前版本，同時完成初始化）必須擇一，兩者不可併用，兩者都不給會直接報錯。

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

報表的 `prunable_artifacts` 是實際計數，與 cleanup dry run 的 `candidate_count` 同口徑（不再固定為 0）。確認盤面已全部搬到目前版本後，再執行 cleanup 的 dry run：

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

## 7. 相關環境變數

- `DATA_CONFIG_VERSION_PATH`：啟動時記錄設定版本用的 `version.json` 路徑，未設定時看工作目錄下的 `config/data/version.json`。檔案不存在只記警告，檔案存在但內容損壞會擋住啟動。

## 8. 排程換版（effective_at）

要讓新配置在指定時間才生效，用 PJA-Config 建置時帶 `-EffectiveAt`（`Build-All.ps1`／`Generate-Version.ps1`，任何 `DateTimeOffset` 可解析的時間，會轉成 UTC 寫入 `version.json` 的 `effective_at`），再照第 1 節發布：

```powershell
.\Scripts\Build-All.ps1 -EffectiveAt "2026-11-01T04:00:00+08:00"
```

- 生效前沿用父 release：`current.json` 已指向新版本，但 World 在 `effective_at` 之前繼續載入 `parent_config_version`；`pkg/dataconfig.LoadActiveSnapshot` 與熱重載（`errReloadScheduled`）都走同一判斷。到點後 watcher 的計時器觸發 reload 換版，不需再發布。
- 以 Central 校正時鐘判斷：World 用 `App.GameNow`（Central TimeSync 校正後的遊戲時鐘），不用主機時鐘，多台主機才會同一時刻換版。
- World 首次 TimeSync 前一律視為未生效、不換版（主機時鐘偏快會讓這台提早換版）；監看 goroutine 對時前不排計時器，首次對時後重跑 reload：該換馬上換，沒到就以校正時間重設計時器（commit `d232bc76e`）。
- 首次發布（沒有父 release）不可設定未來時間：無從退讓，載入會回 `future effective config requires a parent release`。排程換版只用於已有父版本的後續發布。
- 回退仍走第 4 節流程；排程中的版本想取消，回退到父版本或重新發布不帶 `-EffectiveAt` 的版本。

行為與熱重載的交會見[配置熱重載](配置熱重載.md)。
