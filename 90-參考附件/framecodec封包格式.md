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
| `msgID` | 2 位元組 | `uint16`，小端序 | 標準 `MessageID`；Client 端來源為 `PJA-Protocol`，S2S 來源為 Server 註冊表 |
| `payload` | `size - 6` 位元組 | 原始位元組 | 訊息內容，通常是結構序列化後的二進位資料 |

> 所有多位元組整數一律使用**小端序**。

## 欄位詳解

### size（4 位元組）

- 值為**整包長度**（含自身 4 位元組與 msgID 2 位元組），不是 payload 長度。
- 最小合法值為 `6`（空 payload）。
- 最大合法值為 `maxMsgSize = 4 MB`（`4 * 1024 * 1024`，含標頭）。
- 收到 `size < 6` 或 `size > 4 MB` 的封包視為壞包／惡意封包，Server 會直接斷線。

### msgID（2 位元組）

- 區分訊息種類，例如：
  - `10001` = 登入請求（User → Gate）
  - `30102` = S2S 心跳請求（`HeartbeatReq`）
  - `30103` = S2S 心跳回應（`HeartbeatRes`）
  - `30403` = S2S 頻道廣播（`ChannelBroadcast`，World → Gate）
  - `30404` = S2S 頻道成員同步（`ChannelMembershipUpdate`，World → Gate）
  - `30405` = S2S 頻道成員快照（`ChannelMembershipSnapshot`，World → Gate）
  - `10101` = Client 心跳請求（`HeartbeatRequest`）
  - `10102` = Client 心跳回應（`HeartbeatResponse`）
  - `11151` = 加入聊天房間
  - …完整清單見 `PJA-Protocol/Datas/__enums__.xlsx`；路由中繼資料見 `PJA-Protocol/Datas/__beans__.xlsx`
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

再以 `payload = "hello"`（5 位元組）為例（對應 `ExampleBuildFrame`）：

```
frame  = 0B 00 00 00  96 75  68 65 6C 6C 6F
size=11      id=30102  "hello"
```

## Client 端收包建議流程

TCP 是串流，一次 `Read` 不一定剛好拿到一個完整 frame，需自行組合 frame：

1. 讀滿 4 位元組得 `size`。
2. 檢查 `6 <= size <= 4 MB`，不合法直接斷線。
3. 讀滿剩餘 `size - 4` 位元組（含 msgID 與 payload）。
4. 解析 `msgID`（位元組 4～5，小端序）與 `payload`（從位元組 6 開始）。

送包流程：

1. `size = 6 + len(payload)`。
2. 依序寫入 `uint32(size)`（小端序）、`uint16(msgID)`（小端序）、`payload`。
3. 可一次 `Write` 完整 frame，避免半包。

## 相關 API（`pkg/framecodec`）

| API | 用途 |
|-----|------|
| `BuildFrame(msgID, payload, frame)` | 組出一個完整 frame（可傳入既有 buffer 重用） |
| `Coder.Encode(msgID, payload)` | 組 frame，等價於 `BuildFrame` |
| `Coder.EncodePooled(msgID, payload)` | 組 frame 並回傳釋放函式（由物件池管理） |
| `Coder.WriteFrame(w, msgID, payload)` | 直接寫入 `io.Writer` |
| `Coder.Decode(reader, buf)` | 從 reader 讀出一個 frame，回傳 `msgID` 與 `payload` |
| `Coder.DecodePooled(reader)` | 物件池版本解碼 |

`tcpsocket.Coder` 即 `framecodec.Coder` 的別名，Server／Client 連線預設使用本格式。

## 限制摘要

- 單條封包最大 **4 MB**（含 6 位元組標頭），超過即斷線。
- `msgID` 為 `uint16`，最大值 `65535`。
- 整數一律使用小端序。
- 沒有校驗碼／加密；完整性靠 TCP 傳輸層，安全性由上層協定負責。
