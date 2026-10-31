# RabbitMQ 發布可靠性

World 發送 Record 事件時，`pkg/recordmq.Publisher` 會啟用 RabbitMQ publisher confirm：

```text
PublishWithDeferredConfirmWithContext
        ↓
RabbitMQ ack / nack
        ↓
Publish 回傳成功 / 錯誤
```

`PublishWithDeferredConfirmWithContext` 成功只代表訊息已寫入連線；publisher confirm 才代表 RabbitMQ 已接受該訊息。發布端會等待 confirm，收到 `nack`、channel 關閉或 context 取消時回傳錯誤並增加發布錯誤指標。

發布端只宣告 exchange（`exchange`／`exchange_type`），不宣告佇列也不綁定；佇列與綁定鍵（`rabbitmq.binding_key`，舊名 `routing_key` 保留一版相容）由 Record 消費端宣告。空事件類型在 `Begin` 直接回一般錯誤（`recordmq: empty event type`）不送出，outbox 則以 `ErrRecordInvalid` 於領取前擋掉。發布一律帶 `mandatory`，並以 `NotifyReturn` 接收退回：broker 對無法路由的訊息會先回 return 再回 ack，發布端在 confirm 到達後認領退回通知，視為發布失敗，計入 `recordmq_publish_error_total{reason="returned"}`（Record 尚未啟動、佇列不存在時事件不會被 broker 確認後靜默遺失，一般事件記 Warn 丟棄，outbox 事件留在 outbox 依退避重試）。認領機制：每筆發布從 `Publisher` 的原子計數器取一個唯一序號，放進訊息 header `x-publish-seq`（broker 退回時 header 原樣帶回），送出前先 `register` 到目前這組連線與 channel（`amqpLink`）的 `pending` 表；每個 `amqpLink` 各有一把 `returnMu`，保護「排空退回通知並記到對應序號」與「讀取這筆是否被退回」在同一個臨界區，等待者 A 不會讀走 B 的退回而讓 B 排空到空 channel 誤判成功；confirm 到達後以 `wasReturned(seq)` 查結果，成功、逾時、取消、nack 都 `release` 刪除登記（順手排空，避免沒人等待時 256 筆緩衝塞滿而卡住連線 reader），不在表內的退回通知（等待者已放棄）直接丟棄。重連會換上新的 `amqpLink`（新的 `pending` 與退回 channel），舊連線上尚未認領的退回隨舊 link 丟棄，那些等待中的發布因 channel 關閉本來就回錯誤。空事件類型在 `Begin` 回 `ErrPermanent`，不再備援到綁定鍵。

Record 消費端則使用手動 ACK：事件成功交給 handler 後才 ACK，解析失敗會 NACK 並丟棄壞訊息。兩者用途不同：publisher confirm 保護 `World → RabbitMQ`，consumer ACK 保護 `RabbitMQ → Record`。

## 目前的發布路徑

- 每筆發布各自取得 `DeferredConfirmation`：發布鎖只涵蓋送出呼叫，等待 confirm 時已放開，多筆事件可同時等待 broker 確認；單筆最多等 5 秒，逾時視為失敗。連線中斷時發布端在背景重連，退避 1 秒倍增到 30 秒、抖動 ±30%（空窗上限約 40 秒），與 World record sink、tcpclient 一致；`Close` 會取消並等待背景重連 goroutine，只呼叫 `Close` 的使用者不會留下持續撥號的 goroutine。
- 一般事件（`App.PublishRecord`）走 `AsyncPublisher`：事件先進記憶體佇列（容量取 `rabbitmq.buffer_size`，未設定時用 `pkg/recordmq` 的預設 4096 筆；正式與開發設定都寫 4096），由背景 goroutine 逐筆發布並等待 confirm。佇列滿或發布失敗只記 Warn（發布失敗每 10 秒最多一行，附累計失敗數），事件丟棄，不影響遊戲主流程。**broker 斷線期間發布迴圈暫停出列**（每 200ms 重查連線狀態）：事件留在佇列等重連，不會逐筆送進已關閉的 channel 後丟棄；斷線久了佇列填滿，新事件才以 `ErrQueueFull` 被拒。排隊階段的丟棄計入 `recordmq_async_dropped_total{reason}`（`queue_full` 背壓、`stopped` 已進入關機、`not_ready` 已設定 RabbitMQ 但發布器尚未建立；序列事先建立，`increase()` 看得到第一次；`queue_full`／`not_ready` 觸發 `GameRecordEventsDropped` 告警）。每筆事件各自 recover，序列化或 tracing 的一次 panic 只丟該筆，發布迴圈不會永久停止。World 的 `PublishRecord` 以 `context.WithoutCancel` 排隊（保留 trace 值），呼叫端 ctx 取消（Central 重連、關機）不再讓已完成的狀態少一筆稽核事件。關機時先排空佇列，關機期限到了就放棄剩餘事件。
- 例外：GM 發放道具（`item.grant`，`cmd/world/internal/services/grantitem.go`）走上面一般的 fire-and-forget `PublishRecord`，不經 outbox；sink 未就緒（`not_ready`）或佇列滿（`queue_full`）時事件可能遺失，只計入 `recordmq_async_dropped_total`。事件 ID 取 `OperationID`、發生時間取指令的 `RequestedAtMs`（舊指令沒有請求時間時才沿用目前遊戲時間），所以重跑跨 UTC 午夜也不會落到另一張日表而留下兩筆；指令因 ack 失敗等原因重跑時 Record 以 `event_id` 去重，不會留下兩筆發放紀錄。
- 必須送達的事件（經濟、道具等）走 MySQL record outbox：與遊戲異動在同一個交易寫入 outbox，由 relay 每批領取後最多 16 筆同時以 `PublishSync` 發布並等待 confirm，發布成功的再整批一句 `UPDATE` 標記已送出（整批標記失敗退回逐筆）；失敗留在 outbox 依退避時間重試（重試規則在 relay：以領取時的 attempts 逐筆算好次數、延遲與是否死信 `RecordOutboxRetryPlan`，store 的 `MarkRecordOutboxRetries` 只以一句 UPDATE 寫入，不再用 SQL 重寫退避規則；`dead_letter` 指標只由 relay 計），超過重試上限標記 `dead_lettered_at`，payload 不是單一合法 JSON 值（`json.Valid`，含尾隨資料）時直接標記死信、不再退避；合法 payload 以 `json.RawMessage` 原樣發布，不先解成物件樹再編回（`null` 視為無 payload）：`outbox.Record.Payload` 本身就是 `json.RawMessage`，直接帶入 Envelope 的 `Payload`，不經 `SetPayload` 再編碼一次；事件 ID 由 `recordevent.NewEnvelopeWithID` 直接用 OperationID，不再先產生一個隨機 ID 又丟棄。領取租約、下次嘗試時間（含新事件的第一次嘗試時間）與各時間欄位一律以資料庫 `NOW(3)` 計算，多台 World 主機時鐘偏差不影響。標記時租約已被其他 World 取走的列不計送出也不重試，計入 `game_record_outbox_total{result="lease_lost"}`（`result` 另有 `sent`、`retry`、`dead_letter`；重試標記寫入成功才計 `retry`）。版控內沒有死信重送工具與程式 API（2026-10-30 決定刪除無呼叫端的 ReplayRecordOutbox、ListDeadLetterRecordOutbox）；需要重送時以 SQL 把 `record_outbox_event` 該列的 `dead_lettered_at` 清空、`attempts` 歸 0、`next_attempt_at` 設為現在並清掉 `lease_owner`／`lease_until`，且 WHERE 必須加 `created_at >= UTC_DATE() - INTERVAL 30 DAY`（Record 最早接受今天往前 `LookbackDays-1`＝30 天的 UTC 00:00；接近 UTC 午夜執行時再留一天餘裕，例如改用 `INTERVAL 29 DAY`，避免邊界日的列在重送途中越過窗口）：事件 `occurred_at` 固定為 outbox 列建立時間，超過 Record 可寫入窗口時 Record 會判為 `ErrOccurredAtOutOfRange` 送進 MQ 死信，World 卻已標為 sent，事件會靜默遺失；超過窗口的死信不可重送，只能人工處理。沒有設定 RabbitMQ 時不啟動 relay 並記錄警告，outbox 列原樣保留，不會因為發布一律失敗而被燒進死信。有設定 RabbitMQ 時 relay 一律啟動；broker 目前斷線或啟動時尚未連上（`App.IsRecordSinkReady()` 為 false）時 relay 整輪略過：不領取、不發布、不計重試次數，所以 broker 中斷多久都不會把待送事件燒進死信；連線恢復後照常發送。World 啟動時連不上 RabbitMQ 不拒絕啟動，改在背景重試（見下方「啟動連不上 RabbitMQ」），期間以 `recordmq_enabled == 1` 且 `recordmq_connection_status == 0` 觸發 `GameRecordMQDisconnected`。
- 去重：outbox 事件以 operation ID 作為 `event_id`，`occurred_at` 固定為 outbox 列的建立時間，重送跨日時仍落在同一張 Record 日表；Record 寫入時以 `event_id` 去重。

## 消費端重試

批次消費（`RunBatch`）全部成功時，批尾只送一次 `Ack(tag, multiple=true)`（單一 goroutine 依序處理同一 channel 的投遞，本批之前的投遞都已 ack／nack，所以只確認本批），省下逐筆 ack 的 frame 寫出與 flush；整批失敗退回逐筆處理時維持 `multiple=false`，避免連帶確認其他投遞。

Record handler 回傳暫時性錯誤時，消費端 NACK 並放回佇列，第 n 次失敗後暫停 1 秒 × 2^(n-1)（上限 30 秒）再取下一筆；基礎設施類錯誤（MySQL 連線或逾時錯誤、日分表尚未建立 `ErrTableNotReady`、ctx 逾時）只放回佇列並退避，不計入 20 次的死信預算（Warn `recordmq: handler failed` 以 `infraAttempts` 欄位顯示次數），所以 MySQL 停機或外部工具遲建分表不會把正常事件打進死信，代價是缺分表的事件阻塞佇列前端直到恢復；其餘錯誤同一事件失敗 20 次或回傳永久性錯誤（例如 `occurred_at` 超出可寫入範圍，或缺少 `OccurredAt`：Record 只採生產者填的事件時間，不以接收時間補）時 NACK 不重排，交給死信佇列。handler panic 會被接住並記 Error `panic recovered`（`msg=recordmq handler`），逐筆處理失敗另記 Warn `recordmq: handler failed`：整批 panic 視為整批失敗、退回逐筆，逐筆 panic 視為永久性錯誤直接進死信。暫停期間整個消費者停止取件：暫時性錯誤多半是下游 MySQL 故障，整體放慢才不會空轉。

消費結果計入 `recordmq_consume_total{result}`：`ok`（已寫入並 ack）、`retry`（暫時性失敗放回佇列）、`dead_letter`（壞格式、永久錯誤、連續失敗達上限或 handler panic，進 `<queue>.dead`）；序列事先建立。`dead_letter` 5 分鐘內有增加就觸發 `GameRecordConsumerDeadLetter`（page）：這些事件不會再寫進資料庫，需看 Warn `recordmq: handler failed`（處理失敗）或 `recordmq: invalid event payload`（內容無法解析；欄位 `messageID`、`bodySHA`＝body SHA-256 前 6 位元組十六進位、`bodyLen`，不放內容）確認原因，修好後把 `.dead` 的內容重送。消費端自己的連線中斷／建立失敗同樣以指數退避重連（1 秒倍增到 30 秒、抖動 ±30%，連上後歸零），Warn 附 `retryIn`；RabbitMQ 長時間故障時不再每秒一行，多個實例也不會同秒重撥。

失敗計數表記失敗次數與最後失敗時間，`depthTicker` 觸發時清掉超過 2 小時（`failureRetention`）未再失敗的鍵（重排後被其他消費者取走或過期的事件不會永久殘留）。`Consumer.Close` 以 `CloseDeadline`（5 秒）關閉連線並連帶關閉 channel，與 `Publisher.Close` 一致：broker 半開或不回 close-ok 時不會卡到 AMQP 心跳判定斷線，Record 關機不會超過 `stop_grace_period`；未 ack 的投遞由 broker 重送。`recordmq_connection_status` 在消費端連線成功設為 1、`Close` 設為 0，`GameRecordMQDisconnected` 只看 `job="record"`。

## 啟動連不上 RabbitMQ

World 設定了 RabbitMQ 但啟動時連不上，**不拒絕啟動，也不永久停用 record sink**：Record 是旁路，不該因 broker 暫時中斷讓 World 起不來，但降級後永不恢復也不對（compose 自動重啟時 broker 可能還沒好）。因此改在背景以指數退避（`cenkalti/backoff`，1 秒倍增到 30 秒，加 ±30% 隨機抖動，多台 World 同時等 broker 時不會同步重撥；不設總時限，不會放棄）持續重試，連上後才把 sink 交給 App。

- 日誌：啟動失敗 Error `world record sink unavailable at startup, retrying in background; Record events are dropped until it connects`，每次重試失敗 Warn `world record sink retry failed`（`next`、`err`），連上 Info `world record sink connected after startup retry`。
- 連上前一般事件（`App.PublishRecord`）因 sink 為 nil 略過；必須送達的 outbox 事件留在 MySQL。有設定 RabbitMQ 時 outbox relay **一律啟動**，sink 尚未連上時整輪略過（不燒重試次數），連上後照常發送；沒設定 RabbitMQ 才不啟動 relay。
- `recordmq_connection_status=0` 因此是暫態，`GameRecordMQDisconnected` 的 5 分鐘門檻會在持續連不上時才響；不必重啟 World。
- 關機流程會停止重試並排空已連上的 sink，連上與關閉同時發生時直接排空關閉，不會洩漏連線。

## 關機與重連

- `AsyncPublisher.Stop(ctx)` 受關機期限約束：期限到即取消排空與等待中的確認（broker 失聯時不會無限久）；World record sink 傳入關機 ctx。
- 同步發布端 `Close` 先以 `CloseDeadline` 有期限地關閉連線，讓卡在 `connection.blocked` 背壓的寫入失敗返回，再等進行中的發布結束；World record sink 的 `publisher.Close` 另以 `appcore.WaitContext` 受關機任務期限約束，broker TCP 卡住時 Cleanup 不會永遠卡住。World 啟動失敗路徑也會帶期限關閉已建立的 record sink。
- `NotifyClose` 在建立連線的 `dialAndSetup` 內登記（回傳 `amqpLink`）：連線交出後才登記時，中間若已斷線 amqp091 會直接關閉通知 channel，重連迴圈誤判為正常關閉而停止重連，之後 `Publish` 一律 `ErrClosed`。
