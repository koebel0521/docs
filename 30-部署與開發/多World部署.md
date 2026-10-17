---
tags: [project, deploy, world, topology]
created: 2026-10-16
---

# 多 World 部署

同一個環境跑多個 World 時的設定規則、維運操作與驗證方式。單 World 部署不受影響。

## 規則

1. **每個 Gate 的 `dial.world` 必須列出所有 World。** Gate 對每個位址各維持一條連線（`tcpclient.WithDialAll()`），各自獨立重連；Gate 用玩家所在的 World ID 找到那條連線轉送。Central 指派新帳號的 World 時只看 World 是否連到 Central 與負載，**不看請求的那個 Gate 連到哪些 World**，所以少列一個 World，被指派到它的玩家就無法從該 Gate 登入。
2. **每個 World 要有不同的 `id`、監聽位址與 `public_addr`**，並且都連到同一個 Central。
3. **帳號的 World 是固定的**（`user.world_id`）。World 掛掉時，綁在它上面的玩家無法改派到別的 World（記憶體狀態與寫回都在原 World）；其他 World 的玩家不受影響。
4. **每個 World 一個獨立的 `game_store.spill_dir`**（關機時寫不進 MySQL 的玩家異動落地目錄），不要共用。
5. 所有 World 共用同一個 MySQL、Redis 與 RabbitMQ。
6. 診斷埠預設依服務類型固定（`pkg/debug/services.go`：World 為 6062）；同一台主機跑第二個 World 時在其設定檔加 `debug.addr`（例如 `":6072"`），否則後啟動的會記錄 `debug server disabled; set debug.addr ...` 錯誤、沒有 `/metrics`、`/ready` 與 pprof（服務照常運作）。

## 設定範例

Gate（`<gateID>.dial.world` 列出全部 World）：

```yaml
2001:
  dial:
    world:
      1001:
        ip: world1:61001
      1002:
        ip: world2:61002
    central:
      1:
        ip: central:60002
```

World 2（與 World 1 不同的欄位）：

```yaml
id: 1002
public_addr: "world2:61002"
game_store:
  spill_dir: /app/spill2

1002:
  listen:
    gate:
      ip: 0.0.0.0:61002
  dial:
    central:
      1:
        ip: central:60001
```

## 維運操作

- **新增 World**：先啟動新 World（連上 Central），再把它加進每個 Gate 的 `dial.world` 並重啟 Gate。Gate 重啟會中斷其上全部玩家（優雅關機 drain 30 秒），要滾動重啟、一次一個 Gate。新 World 在 Central 重連冷卻（30 秒）結束後開始被指派新帳號。
- **計畫性維護 World**：`POST /admin/world/drain`（Lobby admin API）讓新玩家避開該 World，既有玩家不受影響；維護完成用 `/admin/world/undrain`。`GET /admin/world/draining` 看目前排空集合。
- **World 重啟**：不需要重啟 Gate。Gate 的重連退避最長 30 秒，加上 Central 冷卻 30 秒，綁在該 World 上的玩家約 45～60 秒內可重新登入。
- **canary**：Central 的 `TrafficSliceWorldID` 依帳號雜湊把一部分新帳號導到指定 World，同樣避開排空與冷卻中的 World。

## 故障行為

| 情境 | 結果 |
|---|---|
| 一個 World 掛掉 | 只有綁在它上面的玩家失去服務（Gate 進入 30 秒寬限，之後清掉這些玩家）；其他 World 的玩家與延遲不變；新帳號只會被指派到其餘可選的 World |
| 掛掉的 World 重啟 | Gate 自動連回，不需重啟 Gate |
| Gate 少列了某個 World | 被指派到該 World 的新帳號登入失敗（重試也不會成功）；既有綁在它上面的玩家也登入不了 |
| 所有 World 都在排空或冷卻 | 新玩家回 `AuthServiceUnavailable`（可重試）；既有玩家不受影響 |

## 驗證步驟

壓測用的設定與腳本在 `tools/private/`，驗證時依[壓測手冊](壓測手冊.md)的共通注意事項（來源 IP、`RL_PASSWORD`）操作：

1. 啟動兩個 World 與一個列出兩者的 Gate，確認 Gate 日誌有兩筆 `connect`（`name: World`，分別連 `61001` 與 `61002`）。
2. 用新帳號登入 60 個以上（`RL_PASSWORD=… cmdload -n 60 -rounds 2 -prefix <新前綴>`）：預期全部登入成功，Central `user` 表的 `world_id` 分佈在兩個 World。
3. 600 個新帳號常駐（`soakload -n 600 -prefix <新前綴>`），確認 Gate 連線約 600，擲骰 p99 約 3ms。
4. 殺其中一個 World：預期 Gate 連線降到另一個 World 的玩家數（例如 605 → 297），另一個 World 的玩家擲骰延遲不變。
5. 重啟被殺的 World（**不重啟 Gate**）：約 1 分鐘內該 World 的帳號可重新登入。

## 已驗證行為（2026-10-16 第四輪）

- 兩個 Gate × 三個 World 全網狀運作正常，任一端重啟不需重啟另一端。
- 同帳號跨 Gate 重複登入：舊 Gate 收到 `AuthReLogin`、清 session 並通知玩家，但**不關閉 TCP**；同 Gate 重複登入則直接關閉舊 socket。新連線都存活。
- 排空只影響新帳號指派；三個 World 全部排空時新帳號回 `AuthServiceUnavailable`、不建立帳號，既有帳號仍可登入。
- canary 切片命中的新帳號確定性進指定 World；調整需重啟 Central，沒有執行期 API。

細節見[壓測紀錄 2026-10-16 第四輪](../90-參考附件/歷史/壓測紀錄-2026-10-16-第四輪.md)。

## 歷史

2026-10-16 之前，Gate 的 world client 是備援模式：`dial.world` 的多個位址只維持一條連線，World 1 掛掉時改連 World 2，且 World 1 回來後不再連回。兩個 World 時 60 個新帳號只有 27 個能登入。修正與驗證數字見[壓測紀錄](../90-參考附件/歷史/壓測紀錄-2026-10-15.md)第 21、25 步。
