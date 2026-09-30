# Client 與 S2S 協定分離 XML 方案

## 狀態

方案草稿，尚未改動現行生成流程與線上封包格式。

## 目標

- Client 與 S2S payload 統一使用 Luban 編碼。
- Client 與 S2S 使用獨立 XML，避免跨邊界引用與誤生成。
- 保留現有 frame 格式與 Message ID 分區。
- 生成結果可檢查、可同步、可回滾。

## 目前問題

現行協定有兩種 payload 編碼：

- `pkg/msg/board` 使用 Luban `ByteBuf`。
- Auth、Chat、部分 System 與 S2S 訊息仍使用 `packet.Writer`、`packet.Reader` 或手動固定格式。

Client 結構由 `PJA-Protocol` 生成至 `pkg/protocol/client`；S2S Message ID 則由 Server 維護。若直接把兩者合併到同一份 XML，容易造成 S2S 結構被 Client target 輸出，或使外部 Client 依賴內部協定。

## 目標結構

```text
PJA-Protocol/
└─ Datas/
   └─ protocol.xml              # Client 協定

PJA-Server/
└─ Datas/
   └─ s2s-protocol.xml          # S2S 協定
```

生成結果：

```text
PJA-Protocol/Datas/protocol.xml
    → Client target
    → PJA-Protocol 產生 Client Message ID 與結構
    → PJA-Server/pkg/protocol/client/

PJA-Server/Datas/s2s-protocol.xml
    → S2S target
    → PJA-Server 產生 S2S 結構
    → PJA-Server/pkg/protocol/s2s/
```

兩份 XML 可以共用 Luban 型別與 Server runtime，但不共用生成輸出目錄。

## 邊界規則

### Client XML

只放外部 Client 可見的：

- 登入、登出與驗證
- Client heartbeat
- Chat
- Board、Combat 等遊戲訊息
- Client 可使用的 enum、bean 與 Message ID

輸出固定進入：

```text
pkg/protocol/client/
```

### S2S XML

只放服務間可見的：

- ConnectInfo
- S2S heartbeat
- TimeSync
- Gate、Central、World 間的 Auth 流程
- Channel 同步與廣播
- Forward、Audit 及其他內部控制訊息

輸出固定進入：

```text
pkg/protocol/s2s/
```

S2S 結構不可出現在 `pkg/protocol/client`，Client XML 也不可引用 S2S-only 型別。

## Message ID

維持現有責任分工：

| 類型 | ID 來源 | 建議範圍 |
| --- | --- | --- |
| Client | `PJA-Protocol` canonical ledger | 現有 Client 區段 |
| S2S | PJA-Server S2S registry 或 S2S XML 專用 registry | `30000+` |

S2S XML 初期只負責結構與編碼；Message ID 仍由 `pkg/protocol/s2s_message_id.go` 管理，避免一次同時變更 ID 來源與 payload 格式。待生成流程穩定後，才評估是否讓 S2S XML 產生 ID registry。

任何 target 都必須檢查：

- 不得輸出另一邊的結構。
- Client 與 S2S ID 不得重複。
- 生成結果必須可由 `-Check` 驗證。

## Frame 邊界

不修改現有 frame：

```text
[4B size][2B msgID][Luban payload]
```

本方案只替換 payload 編碼。`size`、`msgID`、Little Endian 與 4 MB 封包上限維持不變。

Heartbeat 沒有欄位時可維持零長度 payload，不為了形式增加無意義欄位。

## 生成與同步流程

### Client

沿用 `scripts/sync-protocol.ps1`，但將檢查責任明確化：

1. 由 Client target 產生 Client registry 與結構。
2. 驗證輸出不含 S2S 結構。
3. 同步至 `pkg/protocol/client`。
4. `-Check` 比對生成結果是否過期。

### S2S

新增獨立腳本，例如：

```text
scripts/sync-s2s-protocol.ps1
```

責任：

1. 讀取 `s2s-protocol.xml`。
2. 以 Server target 產生 S2S 結構。
3. 驗證輸出不含 Client-only 結構。
4. 同步至 `pkg/protocol/s2s`。
5. 支援 `-Check`，供 CI 與提交前檢查使用。

## 程式遷移

分階段替換現有 codec：

1. 建立 S2S XML、生成 target、package 與同步檢查。
2. 將 S2S 訊息改為 Luban `ByteBuf` round-trip。
3. 保留舊 codec 測試，新增 golden bytes 測試確認格式。
4. 將 Client Auth、Chat、Heartbeat schema 化。
5. 更新 Client 端 decoder 後，再切換 Server 的 `Pack/Unpack`。
6. 全部切換完成後，才移除不再使用的 `packet.Writer`、`packet.Reader` 路徑。

每一批訊息都必須同時驗證：

- Pack → Unpack round-trip。
- 欄位順序與型別。
- 空值、空集合與最大長度。
- 舊 Client／舊 Server 的相容性策略。

## 相容性策略

Luban 改編碼後，既有非 Luban payload 通常不會自動相容。登入、Chat 與 S2S 應採用以下其中一種策略：

- 直接升級 Client 與所有 Server，切換同一協定版本。
- 在 Message ID 或 protocol version 上分出新版本。
- 短期保留舊 decoder，完成過渡後移除。

不可只替換 Server encoder 而假設舊 Client 可以解析。

## 風險與控制

| 風險 | 控制方式 |
| --- | --- |
| Client 誤收到 S2S 結構 | 兩份 XML、兩個輸出 package、target deny-list |
| 欄位順序改變造成解碼錯位 | XML review、golden bytes、round-trip test |
| Message ID 重複 | Client/S2S registry 交叉檢查 |
| 舊 Client 無法登入 | protocol version 或整體版本切換 |
| 生成碼與 schema 不一致 | `sync-*.ps1 -Check` 納入 CI |
| S2S 遷移範圍過大 | 先新增 target，再逐批替換 message |

## 驗收條件

- Client 與 S2S 各自有唯一 XML 與生成 target。
- `pkg/protocol/client` 不包含 S2S 結構。
- `pkg/protocol/s2s` 不包含 Client 結構。
- 新增的 Client／S2S payload 都能由 Luban 編解碼。
- frame header 格式不變。
- Client 與 S2S Message ID 不重複。
- CI 能發現生成碼過期或錯誤輸出。
- 完成相容性切換前，舊協定仍有明確回退方式。

## 結論

採用兩份 XML、兩個生成 target、兩個 generated package。這比單一 XML 多一份 schema 管理成本，但能直接隔離 Client 與 S2S 邊界，降低誤輸出與錯誤依賴；對目前已有獨立 Client Message ID 與 S2S registry 的專案結構較安全。
