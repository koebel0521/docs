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
- 必須送達的事件（經濟、道具等）走 MySQL record outbox：與遊戲異動在同一個交易寫入 outbox，由 relay 每批領取後最多 16 筆同時以 `PublishSync` 發布並等待 confirm，發布成功的再整批一句 `UPDATE` 標記已送出（整批標記失敗退回逐筆）；失敗留在 outbox 依退避時間重試，超過重試上限標記 `dead_lettered_at`，payload 不是單一合法 JSON 值（`json.Valid`，含尾隨資料）時直接標記死信、不再退避；合法 payload 以 `json.RawMessage` 原樣發布，不先解成物件樹再編回（`null` 視為無 payload）。領取租約、下次嘗試時間（含新事件的第一次嘗試時間）與各時間欄位一律以資料庫 `NOW(3)` 計算，多台 World 主機時鐘偏差不影響。標記時租約已被其他 World 取走的列不計送出也不重試，計入 `game_record_outbox_total{result="lease_lost"}`（`result` 另有 `sent`、`retry`、`dead_letter`、`replay`；重試標記寫入成功才計 `retry`）；死信由 `cmd/world/tools/outboxreplay` 檢查後重送。沒有設定 record sink（RabbitMQ 發布者）時不啟動 relay 並記錄警告，outbox 列原樣保留，不會因為發布一律失敗而被燒進死信。已設定但 broker 目前斷線時（`App.IsRecordSinkReady()` 為 false（原 `RecordSinkReady`）），relay 整輪略過：不領取、不發布、不計重試次數，所以 broker 中斷多久都不會把待送事件燒進死信；連線恢復後照常發送。World 啟動時首次連不上 RabbitMQ 以 Error 記錄（啟動不重試，整個行程期間都不發 Record），並以 `recordmq_enabled == 1` 且 `recordmq_connection_status == 0` 觸發 `GameRecordMQDisconnected`。
- 去重：outbox 事件以 operation ID 作為 `event_id`，`occurred_at` 固定為 outbox 列的建立時間，重送跨日時仍落在同一張 Record 日表；Record 寫入時以 `event_id` 去重。

## 消費端重試

批次消費（`RunBatch`）全部成功時，批尾只送一次 `Ack(tag, multiple=true)`（單一 goroutine 依序處理同一 channel 的投遞，本批之前的投遞都已 ack／nack，所以只確認本批），省下逐筆 ack 的 frame 寫出與 flush；整批失敗退回逐筆處理時維持 `multiple=false`，避免連帶確認其他投遞。

Record handler 回傳暫時性錯誤時，消費端 NACK 並放回佇列，第 n 次失敗後暫停 1 秒 × 2^(n-1)（上限 30 秒）再取下一筆；同一事件失敗 20 次或回傳永久性錯誤（例如 `occurred_at` 超出可寫入範圍，或缺少 `OccurredAt`：Record 只採生產者填的事件時間，不以接收時間補）時 NACK 不重排，交給死信佇列。handler panic 會被接住並記 `recordmq: handler panic`：整批 panic 視為整批失敗、退回逐筆，逐筆 panic 視為永久性錯誤直接進死信。暫停期間整個消費者停止取件：暫時性錯誤多半是下游 MySQL 故障，整體放慢才不會空轉。

失敗計數表記失敗次數與最後失敗時間，`depthTicker` 觸發時清掉超過 10 分鐘未再失敗的鍵（重排後被其他消費者取走或過期的事件不會永久殘留）。`Consumer.Close` 以 `errors.Join` 同時回報 channel 與連線的關閉錯誤。`recordmq_connection_status` 在消費端連線成功設為 1、`Close` 設為 0，`GameRecordMQDisconnected` 只看 `job="record"`。

## 啟動連不上 RabbitMQ

World 設定了 RabbitMQ 但啟動時連不上，**不拒絕啟動，也不永久停用 record sink**：Record 是旁路，不該因 broker 暫時中斷讓 World 起不來，但降級後永不恢復也不對（compose 自動重啟時 broker 可能還沒好）。因此改在背景以指數退避（1 秒倍增到 30 秒）持續重試，連上後才把 sink 交給 App。

- 日誌：啟動失敗 Error `world record sink unavailable at startup, retrying in background; Record events are dropped until it connects`，每次重試失敗 Warn `world record sink retry failed`（`next`、`err`），連上 Info `world record sink connected after startup retry`。
- 連上前一般事件（`App.PublishRecord`）因 sink 為 nil 略過；必須送達的 outbox 事件留在 MySQL。有設定 RabbitMQ 時 outbox relay **一律啟動**，sink 尚未連上時整輪略過（不燒重試次數），連上後照常發送；沒設定 RabbitMQ 才不啟動 relay。
- `recordmq_connection_status=0` 因此是暫態，`GameRecordMQDisconnected` 的 5 分鐘門檻會在持續連不上時才響；不必重啟 World。
- 關機流程會停止重試並排空已連上的 sink，連上與關閉同時發生時直接排空關閉，不會洩漏連線。

## 關機與重連

- `AsyncPublisher.Stop(ctx)` 受關機期限約束：期限到即取消排空與等待中的確認（broker 失聯時不會無限久）；World record sink 傳入關機 ctx。
- 同步發布端 `Close` 先以 `CloseDeadline` 有期限地關閉連線，讓卡在 `connection.blocked` 背壓的寫入失敗返回，再等進行中的發布結束；World record sink 的 `publisher.Close` 另以 `appcore.WaitContext` 受關機任務期限約束，broker TCP 卡住時 Cleanup 不會永遠卡住。World 啟動失敗路徑也會帶期限關閉已建立的 record sink。
- `NotifyClose` 在建立連線的 `dialAndSetup` 內登記（回傳 `amqpLink`）：連線交出後才登記時，中間若已斷線 amqp091 會直接關閉通知 channel，重連迴圈誤判為正常關閉而停止重連，之後 `Publish` 一律 `ErrClosed`。
