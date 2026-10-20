# framecodec 封包格式說明（供 Client 對接使用）

本文件說明 `pkg/framecodec` 的 TCP 封包（frame）打包方式，供 Client（外部對接端）實作收發協定。Server／Client／Socket 之間的連線一律使用此格式。

## 封包總覽

每條 TCP 訊息（frame）由 6 位元組標頭與 payload 組成：

```
[4B size][2B msgID][payload]
```

| 欄位 | 大小 | 型別 / 位元組序 | 說明 |
|------|------|-----------------|------|
| `size` | 4 位元組 | `uint32`，小端序 | 整個封包總長度 = `6 + len(payload)`，含標頭本身 |
| `msgID` | 2 位元組 | `uint16`，小端序 | 訊息 ID；Client 與 S2S 訊息都由 `PJA-Protocol` 產生（見下方 msgID 說明） |
| `payload` | `size - 6` 位元組 | 原始位元組 | 訊息內容，通常是結構序列化後的二進位資料 |

> 所有多位元組整數一律使用**小端序**。

## 欄位詳解

### size（4 位元組）

- 值為**整包長度**（含自身 4 位元組與 msgID 2 位元組），不是 payload 長度。
- 最小合法值為 `6`（空 payload）。
- 最大合法值依連線種類而異（含標頭）：**玩家連線（Client↔Gate）64 KB**（Gate 的 `gateUserMaxFrameSize = 64 * 1024`）；服務間（S2S）連線 4 MB（`maxMsgSize = 4 * 1024 * 1024`）。
- 收到 `size < 6` 或 `size` 超過該連線上限的封包視為壞包／惡意封包，Server 會直接斷線。Client 不可送超過 64 KB 的單一封包。

### msgID（2 位元組）

- 區分訊息種類。號段與常用 ID（例如 Client 登入 `LoginRequest = 1001`、心跳 `HeartbeatRequest = 2005`、聊天 `ChatJoin = 2201`、S2S 心跳 `HeartbeatReq = 30102`）見 [TCP封包參考](TCP封包參考.md) 的號段表。
- 完整清單以 `deps/PJA-Protocol/Datas/protocol.xml` 為準，或查生成碼 `pkg/protocol/wire/Protocol.ProtocolMessageId.go`；Client 與 S2S 訊息都由 PJA-Protocol 產生，沒有另外的 Server 註冊表。
- 型別為 `uint16`，範圍 `0 ~ 65535`。

### payload（`size - 6` 位元組）

- 訊息本體，依 `msgID` 對應的 DTO 結構序列化成的位元組。
- 可以是空（`size == 6`）。

## 範例

以 `msgID = 2005`（Client 心跳請求）、`payload` 為空位元組為例：

```
size   = 6 + 0 = 6 (0x06)
frame  = 06 00 00 00  D5 07
         └─ size ─┘   └id┘
         （小端序 uint32）  （小端序 uint16）
```

再以 `payload = "hello"`（5 位元組）為例（對應 `ExampleCoder_WriteFrame`）：

```
frame  = 0B 00 00 00  96 75  68 65 6C 6C 6F
size=11      id=30102  "hello"
```

## Client 端收包建議流程

TCP 是串流，一次 `Read` 不一定剛好拿到一個完整 frame，需自行組合 frame：

1. 讀滿 4 位元組得 `size`。
2. 檢查 `6 <= size <= 該連線上限`（玩家連線 64 KB、S2S 4 MB），不合法直接斷線。
3. 讀滿剩餘 `size - 4` 位元組（含 msgID 與 payload）。
4. 解析 `msgID`（位元組 4～5，小端序）與 `payload`（從位元組 6 開始）。

送包流程：

1. `size = 6 + len(payload)`。
2. 依序寫入 `uint32(size)`（小端序）、`uint16(msgID)`（小端序）、`payload`。
3. 可一次 `Write` 完整 frame，避免半包。

## 相關 API（`pkg/framecodec`）

| API | 用途 |
|-----|------|
| `Coder.WriteFrame(w, msgID, payload)` | 組出 frame 並以一次 `Write` 寫入 `io.Writer` |
| `Coder.WriteFramePrebuilt(w, msgID, frame)` | frame 已預留 header 時只填 header 後寫出（Socket 傳送路徑使用） |
| `Coder.Decode(reader, buf)` | 從 reader 讀出一個 frame，回傳 `msgID` 與 `payload` |
| `Coder.DecodePooled(reader)` | 物件池版本解碼 |

Socket／OwnerBase 直接持有 `*framecodec.Coder`（`pkg/tcpsocket` 已沒有 `Coder` 別名），Server／Client 連線預設使用本格式。

## 限制摘要

- 單條封包最大：玩家連線 **64 KB**、S2S 連線 4 MB（皆含 6 位元組標頭），超過即斷線。
- `msgID` 為 `uint16`，最大值 `65535`。
- 整數一律使用小端序。
- 沒有校驗碼／加密；完整性靠 TCP 傳輸層，安全性由上層協定負責。
