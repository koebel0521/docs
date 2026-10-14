# RabbitMQ 發布可靠性

World 發送 Record 事件時，`pkg/recordmq.Publisher` 會啟用 RabbitMQ publisher confirm：

```text
PublishWithContext
        ↓
RabbitMQ ack / nack
        ↓
Publish 回傳成功 / 錯誤
```

`PublishWithContext` 成功只代表訊息已寫入連線；publisher confirm 才代表 RabbitMQ 已接受該訊息。發布端會等待 confirm，收到 `nack`、channel 關閉或 context 取消時回傳錯誤並增加發布錯誤指標。

Record 消費端則使用手動 ACK：事件成功交給 handler 後才 ACK，解析失敗會 NACK 並丟棄壞訊息。兩者用途不同：publisher confirm 保護 `World → RabbitMQ`，consumer ACK 保護 `RabbitMQ → Record`。

## 目前的發布路徑

- 每筆發布各自取得 `DeferredConfirmation`：發布鎖只涵蓋送出呼叫，等待 confirm 時已放開，多筆事件可同時等待 broker 確認；單筆最多等 5 秒，逾時視為失敗。連線中斷時發布端在背景重連。
- 一般事件（`App.PublishRecord`）走 `AsyncPublisher`：事件先進記憶體佇列（World 預設 100 筆），由背景 goroutine 逐筆發布並等待 confirm。佇列滿或發布失敗只記 Warn，事件丟棄，不影響遊戲主流程。關機時先排空佇列，關機期限到了就放棄剩餘事件。
- 必須送達的事件（經濟、道具等）走 MySQL record outbox：與遊戲異動在同一個交易寫入 outbox，由 relay 每批領取後最多 16 筆同時以 `PublishSync` 發布並等待 confirm，發布成功的再整批一句 `UPDATE` 標記已送出（整批標記失敗退回逐筆）；失敗留在 outbox 依退避時間重試，超過重試上限標記 `dead_lettered_at`，payload 無法解碼時直接標記死信、不再退避。領取租約、下次嘗試時間（含新事件的第一次嘗試時間）與各時間欄位一律以資料庫 `NOW(3)` 計算，多台 World 主機時鐘偏差不影響。標記時租約已被其他 World 取走的列不計送出也不重試，計入 `game_record_outbox_total{result="lease_lost"}`（`result` 另有 `sent`、`retry`、`dead_letter`、`replay`；重試標記寫入成功才計 `retry`）；死信由 `cmd/world/tools/outboxreplay` 檢查後重送。沒有設定 record sink（RabbitMQ 發布者）時不啟動 relay 並記錄警告，outbox 列原樣保留，不會因為發布一律失敗而被燒進死信。
- 去重：outbox 事件以 operation ID 作為 `event_id`，`occurred_at` 固定為 outbox 列的建立時間，重送跨日時仍落在同一張 Record 日表；Record 寫入時以 `event_id` 去重。

## 消費端重試

Record handler 回傳暫時性錯誤時，消費端 NACK 並放回佇列，第 n 次失敗後暫停 1 秒 × 2^(n-1)（上限 30 秒）再取下一筆；同一事件失敗 20 次或回傳永久性錯誤（例如 `occurred_at` 超出可寫入範圍，或缺少 `OccurredAt`：Record 只採生產者填的事件時間，不以接收時間補）時 NACK 不重排，交給死信佇列。handler panic 會被接住並記 `recordmq: handler panic`：整批 panic 視為整批失敗、退回逐筆，逐筆 panic 視為永久性錯誤直接進死信。暫停期間整個消費者停止取件：暫時性錯誤多半是下游 MySQL 故障，整體放慢才不會空轉。
