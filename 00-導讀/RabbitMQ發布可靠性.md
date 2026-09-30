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

目前發布端為單筆等待 confirm，避免確認 channel 與發布順序失配；若未來事件量需要提升，再改成批次／非同步 confirm，並同步加入事件重試與冪等去重。
