---
tags: [performance, gc, go]
created: 2026-09-14
---

# Go GC 審查紀錄

## 結論

目前沒有證據支持全面新增 `sync.Pool`。專案已有 frame buffer、message pool 與固定持久化 worker pool；優化應以 benchmark、`alloc_objects` profile 為依據，避免把配置轉成長期保留的記憶體。

## 已完成修正

`pkg/logger.IsDebugEnabled` 原本呼叫 `getLogger().Debug().Enabled()`，即使 Debug 未啟用也會建立 zerolog event。改為直接檢查 logger level：

```go
return getLogger().GetLevel() <= zerolog.DebugLevel
```

send path benchmark 結果：

| 指標 | 修正前 | 修正後 |
| --- | ---: | ---: |
| 配置數 | 9 allocs/op | 7 allocs/op |
| 配置量 | 1312 B/op | 688 B/op |
| 執行時間 | 約 480 ns/op | 約 267 ns/op |

提交：`bfa17cac perf: 降低 debug level 檢查配置`

## 玩家資料持久化

主要配置點位於 `pkg/playerstoreredis`：

- `json.Marshal(state)` 每次建立 JSON buffer。
- `string(b)` 將 JSON payload 轉成 Redis script 所需的字串。
- `SaveBatch` 每批 500 筆會建立 `[]string` 與每筆的 ID、World ID、JSON、TTL 參數。
- `RestoreByWorld` 會配置 entries、Redis 回傳 bytes 與 JSON decoder。

基準結果：

| 測試 | 配置量 | 配置數 |
| --- | ---: | ---: |
| 單筆 `json.Marshal + string` | 120 B/op | 3 allocs/op |
| `bytes.Buffer + Encoder` 重用 | 72 B/op | 2 allocs/op |
| 500 筆批次參數組裝 | 約 94 KB/op | 1902 allocs/op |

`Encoder` 重用不能直接套用到 `SaveBatch`：批次必須保留每筆 payload 到 Redis script 執行，重用同一個 buffer 會覆蓋前一筆資料。

曾驗證 `RunBinary(...any)` 搭配 rueidis `BinaryString` 的跨層方案，但 interface boxing 使批次 benchmark 惡化至約 3402 allocs/op，已撤回。

基準提交：`ac3ea440 test: 新增玩家持久化配置基準`

## 訊息與 TCP 路徑

目前 benchmark：

- `pkg/tcpsocket` `PackMsg`：56 B/op、2 allocs/op。
- `pkg/tcpsocket` `WriteFrame`：24 B/op、1 alloc/op。
- `pkg/tcpsocket` `SendPath`：約 688 B/op、7 allocs/op（Debug 日誌啟用時）。
- `pkg/msg` `LoginRequest`：80 B/op、3 allocs/op。
- `pkg/msg` `LoginGateToCentral`：248 B/op、5 allocs/op。

判斷：

- 一般 `Pack()` 使用 `PackTo(nil)`，配置是為了讓非同步送出期間 payload 不受訊息物件回收影響，不能直接改成共用訊息內部 buffer。
- 實際 send path 已有 `PackTo` 與 `WriteFramePrebuilt` 快路徑，應優先使用，不應再加一層 pool。
- Debug 開啟時的 zerolog event 配置是可預期成本；若要消除，需接受降低熱路徑 debug 日誌細節。

## 玩家變數與資源流程（2026-09-14）

本輪審查範圍：玩家變數同步、資源扣除 outbox payload、變數狀態協定解碼。

已完成：

- `variable.Service.Add/Set/Flush` 不在全域 mutex 內執行 DB I/O，改用固定 64 條 striped locks，避免慢查詢串行阻塞所有玩家。
- 變數狀態解碼使用 backing array，避免每筆 `VariableStateData` 個別配置。
- 資源 outbox payload 使用具名 struct，移除每筆 `map[string]any` 的配置與反射型欄位組裝。

基準結果（Windows amd64、AMD Ryzen 5 9600X）：

| 測試 | 執行時間 | 配置量 | 配置數 |
| --- | ---: | ---: | ---: |
| `BenchmarkResourceConsumePayload` | 約 1.26 µs/op | 1056 B/op | 3 allocs/op |
| 舊版 `map[string]any` 對照 | 約 7.1 µs/op | 2668 B/op | 144 allocs/op |
| `BenchmarkReadVariableStates` | 約 93–102 ns/op | 384 B/op | 2 allocs/op |

驗證：

```powershell
go test -benchmem -bench . -count=5 ./cmd/world/internal/store/gamestore ./pkg/msg/board
go test -race ./cmd/world/internal/variable ./cmd/world/internal/store/gamestore ./pkg/msg/board
go test ./...
```

以上三項均通過。相關提交：`a6ac7635`、`bc0bcdc6`、`8addd81a`。

## 後續規則

1. 先跑 `go test -benchmem -count=10`，再決定是否修改。
2. 使用 `alloc_objects` profile 找共同分配來源，不以 `make` 或 `json.Marshal` 的表面位置猜測。
3. 只有在 buffer 大小穩定、生命週期明確且 benchmark 證明有效時才使用 `sync.Pool`。
4. 優先使用既有 `PackTo`、`WriteFramePrebuilt`、buffer reuse 與固定 worker pool。

## EncodePooled 審查

`pkg/framecodec.Coder.EncodePooled` 回傳 `release func()`，該 closure 會造成額外配置。拆分 benchmark 結果：

| 路徑 | 配置量 | 配置數 | 執行時間 |
| --- | ---: | ---: | ---: |
| `EncodePooled` | 56 B/op | 2 allocs/op | 約 34–36 ns/op |
| 直接 Get / BuildFrame / Put | 24 B/op | 1 alloc/op | 約 23–24 ns/op |

目前 `EncodePooled` 沒有 production caller，只有 benchmark 使用；實際 socket 路徑已直接使用 buffer 取得、`BuildFrame`、`WriteFramePrebuilt` 與歸還流程。因此不新增替代 API，也不為此修改公開介面。

## RestoreByWorld 實測

補測 500 玩家恢復流程：

```text
約 6.5 ms/op
約 590 KB/op
11967 allocs/op
```

這是啟動或恢復批次的短時 GC spike 候選，但不是一般遊戲 tick 熱路徑。若恢復延遲或 GC pause 成為實際問題，再優先考慮重用 `entries`/批次暫存與低配置 JSON 解碼；目前不提前引入 pool。
