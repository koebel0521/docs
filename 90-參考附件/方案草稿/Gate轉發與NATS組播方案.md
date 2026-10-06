# Gate 轉發與 NATS 組播方案

> 狀態：初步設計草稿，尚未實作；現行行為以 [聊天服務](../../10-核心流程/聊天服務.md) 與 [服務流程架構圖](../../00-導讀/服務流程架構圖.md) 為準。
> 目標：統一 Gate 的轉發寫法，讓新增功能不必修改 Gate，並以第三方庫取代自建的跨服與組播機制。
> 開發期不受「最小改動」限制，名稱可重新命名。

## 現況問題

以下皆由程式碼確認，除非另有標註。

1. **Gate 上行有三種轉發寫法**
   - 盤面：不解析，原樣包進 `ClientBoardPayload`（`pkg/msg/board/forward.go`）。
   - `RollDice`：Gate 先解包，再 `Pack()` 重新打包轉送（`cmd/gate/internal/handler/user/board.go`）。
   - 聊天：Gate 解析後以同型別重建（`Join`、`Leave`、`Send`、`Sync` 各一個 handler）。
   - 新增功能就要改 Gate 程式。
2. **下行有三種寫法**：`BoardClientEvent`（單播）、`ChatDeliver`（Chat 逐人送）、`ChannelBroadcast`（每個 Gate 一份）。
3. **Chat 多實例可能重複處理（待實測）**：Gate 的 `onChatSend` 以 `chatSvc.SendAll(fwd)` 送給所有 Chat 實例；每個實例都會 `AppendRecord`，並對「所有 Gate × 所有成員」發 `Deliver`。部署 N 台 Chat 時可能重複寫入、重複發送。此點只由程式推得，未實測。
4. **Chat 發送量大**：每則訊息送出「Gate 數 × 成員數」份 `Deliver`，且每份都帶完整 Payload；Chat 沒有記錄玩家在哪台 Gate。
5. **跨服轉發要經 Central 中繼**：`ForwardToWorld` 為 World → Central → World。
6. **成員表兩套**：World 以 `MembershipUpdate` / `MembershipSnapshot` 同步給 Gate；Chat 自己另有一套記憶體成員表，兩者不共用。
7. **滿載策略不一致**：`PushMsgEvent` 在 `eventCh` 滿時關閉連線；`OwnerBase.enqueue`（`SendAll` / `Send` / `SendByID`）滿載時靜默丟棄並只記指標。

## 設計原則

- 順序是「路徑」的屬性，不是 channel 的屬性。需要互相保序的訊息，必須有唯一發送者，並走同一條連線或同一個 subject。
- Gate 只處理自己的訊息（登入、heartbeat），其餘 Client 訊息一律透傳，不解析。
- 玩家狀態相關訊息（盤面、移動、道具）維持 World → Gate 的單一 TCP 路徑，不走 NATS。
- NATS 只用於沒有與玩家狀態保序需求的場景：Chat 組播與跨 World S2S。

## 新設計

### 上行：`ClientForward`

取代 `ClientBoardPayload` / `ClientBoardCommand`、聊天的上行轉發與 `RollDice` 特例。

```
Client ──> Gate ──ClientForward{PlayerID, MsgID, Payload}──> 依路由表選後端
```

- 欄位沿用現有結構：`PlayerID + ClientMessageID + Payload`，沒有盤面專屬欄位。
- 路由表由 PJA-Protocol 產生。`protocol.xml` 已有 `receiver` 屬性，產生器輸出 `MsgID → 目標服務`，Gate 查表轉發。新增功能只改 `protocol.xml`。
- 實例選擇：
  - World：依 `user.WorldID`，固定同一台。
  - Chat：依 `hash(RoomID)` 選一台，取代 `SendAll`，同時解決「現況問題 3」。
- 沿用 `maxClientBoardPayload`（1 MiB）上限與 `PackTo` 失敗即關閉連線的行為。

### 單播下行：`ClientDeliver`

取代 `BoardClientEvent`、`ChatDeliver`。只送到玩家所在的 Gate；後端需知道玩家所在 Gate（World 已有 `user.GateID`；Chat 需新增，見「Chat 改造」）。

### 組播

| 發送者 | 路徑 | 理由 |
|---|---|---|
| World | **維持 TCP**：每個 Gate 一份 `ChannelBroadcast`，Gate 依成員投影 fanout | 與 World 的單播共用同一條 TCP，天然有序 |
| Chat | **NATS** | 聊天不需與玩家狀態事件保序，且可省去逐人發送 |

World 的 `ChannelBroadcast`、`MembershipUpdate`、`MembershipSnapshot` 保留。理由見「順序保證」。

### 跨 World S2S：NATS

取代 `ForwardToWorld`，不再經 Central：

- 每台 World 啟動時訂閱 `pja.world.<worldID>.in`。
- 送給 World B：`nc.Publish("pja.world.B.in", data)`。
- 需要回覆時用 `nc.Request(subject, data, timeout)`。

## NATS 使用方式

### 基本概念

| 概念 | 說明 |
|---|---|
| Subject | 以 `.` 分段的字串，不用預先建立 |
| Publish | `nc.Publish(subject, data)`；沒人訂閱就丟棄，不報錯 |
| Subscribe | `nc.Subscribe(subject, handler)`；`*` 比對一段，`>` 比對其後所有段 |
| Queue group | `nc.QueueSubscribe(subject, "chat", handler)`；同組只有一個訂閱者收到，用於負載分攤 |
| 傳遞保證 | Core NATS 為 at-most-once，不持久化，斷線期間訊息會遺失；本方案不用 JetStream |
| 自動重連 | Client 斷線自動重連並重新送出所有訂閱 |
| 訊息大小 | 預設上限 1 MB（`max_payload`） |

### Subject 規劃

```
pja.topic.chat.<roomID>    Chat 組播：Chat 發布，Gate 訂閱
pja.world.<worldID>.in     跨服：送給指定 World
pja.chat.cmd               Chat 上行：Chat 各實例以 queue group "chat" 分攤
```

房間內順序需求見「順序保證」；若要嚴格依房間保序，`pja.chat.cmd` 改為 `pja.chat.cmd.<分片號>`，分片號為 `hash(roomID) % 分片數`，每個分片只由一個實例訂閱。

### 訊息格式

Payload 前加固定二進位前綴，不使用 NATS header：

```
| MsgID uint16le | SenderPlayerID uint64le | Payload |
```

Gate 切出前 10 bytes，其餘直接 `msg.NewRaw(MsgID, Payload)` 送給玩家。NATS header 只用於 OpenTelemetry trace 傳播，不放業務欄位。

### Gate：依興趣訂閱

規則：本地有玩家在某 topic 時才訂閱；第一位本地成員加入時 `Subscribe`，最後一位離開時 `Unsubscribe`。

| 情境 | 誰通知 Gate |
|---|---|
| 聊天房間 Join / Leave | Client 請求經 Gate 轉給 Chat；Chat 同意後回覆，Gate 再訂閱 |
| Server 主導的變更（公會、隊伍） | 後端以單播送 `TopicJoin{PlayerID, Topic}` 給玩家所在的 Gate |
| 玩家斷線 | Gate 自行清除，不通知任何人 |

NATS 重連後訂閱自動恢復，Chat 的組播不需要 `MembershipSnapshot`。

### 回呼必須進入 event loop

Gate 的所有 handler 在單一 event loop 內依序執行（見「已驗證的事實」）。NATS 的訂閱回呼在 NATS 自己的協程上執行，若直接 `SendMsg` 並操作 `TopicHub` 的 map，會與 TCP handler 並行。因此：

- NATS 回呼只做一件事：把處理封裝成事件，推入 `app.EventCh()`（`appcore.NewFnEvent`）。
- `TopicHub` 的 map 只在 event loop 內存取，不需加鎖。
- `eventCh` 滿載時，`PushMsgEvent` 會關閉連線；NATS 回呼推送事件時，要先決定滿載策略（丟棄並計入指標，優於關閉玩家連線），並在實作時明確寫出。

`TopicHub` 草稿（event loop 版）：

```go
// TopicHub 依本地成員數決定是否訂閱 NATS subject；所有方法只在 event loop 內呼叫。
type TopicHub struct {
	nc     *nats.Conn
	events chan<- appcore.Event
	users  tcpruntime.Service
	topics map[string]*topicSub
}

type topicSub struct {
	sub     *nats.Subscription
	members map[uint64]struct{}
}

func (h *TopicHub) Join(topic string, playerID uint64) error {
	t := h.topics[topic]
	if t == nil {
		sub, err := h.nc.Subscribe(topic, func(m *nats.Msg) {
			data := m.Data
			// 回呼在 NATS 協程，改交給 event loop 執行；滿載時丟棄
			select {
			case h.events <- appcore.NewFnEvent(func() { h.fanout(topic, data) }):
			default:
			}
		})
		if err != nil {
			return err
		}
		t = &topicSub{sub: sub, members: make(map[uint64]struct{}, 4)}
		h.topics[topic] = t
	}
	t.members[playerID] = struct{}{}
	return nil
}

func (h *TopicHub) Leave(topic string, playerID uint64) {
	t := h.topics[topic]
	if t == nil {
		return
	}
	delete(t.members, playerID)
	if len(t.members) == 0 {
		_ = t.sub.Unsubscribe()
		delete(h.topics, topic)
	}
}

func (h *TopicHub) fanout(topic string, data []byte) {
	t := h.topics[topic]
	if t == nil || len(data) < 10 {
		return
	}
	raw := msg.NewRaw(protocol.MessageID{Value: binary.LittleEndian.Uint16(data)}, data[10:])
	for pid := range t.members {
		if sock := h.users.ByID(pid); sock != nil {
			sock.SendMsg(raw)
		}
	}
}
```

上述程式碼未編譯驗證，`m.Data` 的所有權、`raw` 是否可多 socket 共用，實作時要對照 `msg.NewRaw` 與 `Put()` 行為確認（現有 `onChannelBroadcast` 已共用同一個 `raw`）。

### 連線設定

```go
nc, err := nats.Connect(url,
	nats.Name("gate-1"),
	nats.MaxReconnects(-1),
	nats.ReconnectWait(500*time.Millisecond),
	nats.DisconnectErrHandler(func(_ *nats.Conn, err error) { logger.Warn("nats disconnected", "err", err) }),
	nats.ReconnectHandler(func(nc *nats.Conn) { logger.Info("nats reconnected", "url", nc.ConnectedUrl()) }),
	nats.ErrorHandler(func(_ *nats.Conn, s *nats.Subscription, err error) { logger.Error("nats async error", "subject", s.Subject, "err", err) }),
)
// 關閉時以 nc.Drain() 先處理完已收到的訊息
```

日誌一律使用專案的 zerolog `logger`。

### 部署與監控

```yaml
nats:
  image: nats:2-alpine
  command: ["-m", "8222"]
  ports: ["4222:4222", "8222:8222"]
```

- 正式環境用 3 節點叢集（`--cluster` 加 `--routes`），Client 連線字串列出全部節點。
- 監控：`prometheus-nats-exporter` 讀取 8222 埠，輸出到現有 Prometheus。
- 權限：依服務限制 publish / subscribe 範圍（例如 Gate 只能訂閱 `pja.topic.>`、發布 `pja.chat.cmd`）。需與 [S2S 身份驗證](../../30-部署與開發/S2S身份驗證.md) 對照，本草稿未讀該文件。

### 測試

測試內可嵌入 NATS server，不需另起 docker：

```go
import natsserver "github.com/nats-io/nats-server/v2/test"

s := natsserver.RunRandClientPortServer()
defer s.Shutdown()
nc, _ := nats.Connect(s.ClientURL())
```

可測情境：訂閱與取消訂閱計數、兩台 Gate 只收到自己有成員的 topic、重連後訂閱恢復、回呼推入 event loop 後不並行。

## 順序保證

### 各路徑

| 路徑 | 保證 |
|---|---|
| 同一條 TCP 連線 | 嚴格有序 |
| NATS：同一發布者、同一 subject | 有序 |
| NATS：不同 subject | 不保證，即使同一發布者 |
| NATS：不同發布者 | 不保證 |
| Chat 多實例 + queue group | 不保證 |

### 已驗證的事實：Gate 內部依序執行 handler

```
各 Socket 讀取協程 ── PushMsgEvent ──> eventCh（每個 App 一條）
                                         └─ runLoop（單一協程）→ drainEvents → ev.Run() → handler
```

- `PushMsgEvent`（`pkg/tcpsocket/socket_async.go`）把訊息放入 `s.eventCh`，不在讀取協程內執行 handler。
- `drainEvents`（`pkg/appcore/app_tick_drain.go`）逐筆呼叫 `ev.Run()`；`runLoop`（`pkg/appcore/app_base_run_loop.go`）是單一 `for` 迴圈。未找到另外啟動多個 `drainEvents` 的地方。
- 單一 socket 的訊息有序；不同 socket 以進入 `eventCh` 的先後為準，但 handler 不會並行。

### 單播（TCP）與廣播（NATS）混用的風險

World 對同一玩家發出的單播（TCP）與廣播（NATS）走兩條獨立路徑，到達 Gate 的先後不保證。例如先單播「擲骰結果」、再廣播「玩家移動到某格」，Client 可能看到順序顛倒。

因此：**World 發出的單播與廣播都走同一條 TCP**（維持現狀）。若未來仍要把 World 廣播改走 NATS，單播也必須走同一個發布者連線與同一個 subject，並由 Gate 依訊息類型分流，代價是 subject 設計變複雜，不建議。

### 分片鍵與序號

- 玩家盤面事件：玩家固定在一台 World，World → Gate 走同一條 TCP。
- 聊天房間：`hash(RoomID)` 固定 Chat 實例，並發布到同一個 subject。
- 跨路徑必須保序時：Payload 帶每位玩家遞增的 `Seq`，Client 排序；這是最後手段。

## Chat 改造

```
Client --Send--> Gate --ClientForward(hash(RoomID))--> 對應 Chat 實例
Chat：驗證 → 寫 MySQL 取得 MessageID → Publish pja.topic.chat.<roomID>（一次）
NATS --> 有該房間成員的 Gate --> 本地成員
```

- 以 `hash(RoomID)` 選實例取代 `SendAll`，每則訊息只由一台處理。
- 每則訊息只發布一次，取代「Gate × 成員」逐筆送出。
- 遺失由 `ChatSync` 補回（Core NATS 為 at-most-once）；Client 依 `MessageID` 去重與排序。
- 需新增：Chat 在 Join 時記錄玩家所在 Gate，供單播（`ClientDeliver`）與同步回覆使用。

## 命名

| 舊 | 新 |
|---|---|
| `ClientBoardPayload`、`ClientBoardCommand` | `ClientForward` |
| `BoardClientEvent`、`ChatDeliver` | `ClientDeliver` |
| `ForwardToWorld` | 刪除，改用 NATS subject |
| `ChannelBroadcast`、`Channel*` | 保留（World 仍使用） |

新增名詞前須依 [名詞定義](../../../../PJA-Docs/名詞定義.md) 的規定先確認；`Topic` 與 `Channel` 是否並存待決定。

## 風險與取捨

| 項目 | 說明 |
|---|---|
| 新的依賴元件 | NATS 掛掉時 Chat 組播與跨服中斷，需叢集與監控 |
| 斷線期間訊息遺失 | 聊天有 `ChatSync` 補救；其他走 NATS 的訊息若不能遺失，需另行設計補送或改用 JetStream |
| 延遲 | 多一跳 NATS；同機房預期在 1 ms 內，未實測 |
| 慢消費者 | 回呼堆積超過 pending 上限會丟訊息，需以 `sub.SetPendingLimits` 與 `ErrorHandler` 記錄並加指標 |
| 依賴套件 | `github.com/nats-io/nats.go`；測試另加 `github.com/nats-io/nats-server/v2`；加入後以 `govulncheck` 檢查 |

## 遷移順序

1. PJA-Protocol：新增 `ClientForward` / `ClientDeliver`，由 `receiver` 產生路由表，Gate 改為查表。
2. 盤面與 `RollDice` 切到新路徑，刪除 `ClientBoardPayload`。
3. 聊天切到新路徑，Chat 依 `hash(RoomID)` 選實例，修正 `SendAll`。
4. 引入 NATS：先做 Chat 組播（`TopicHub`），再改跨 World S2S，最後刪除 `ForwardToWorld`。
5. 更新 [聊天服務](../../10-核心流程/聊天服務.md)、[服務流程架構圖](../../00-導讀/服務流程架構圖.md)、`名詞定義.md`。

每一步可單獨上線與測試；第 4 步需在 docker compose 加入 NATS 服務。

## 待決定

- 是否引入 NATS。若不引入，第 4 步改為「Chat 採用 World 的成員表同步做法」。
- 是否要先驗證現況問題 3（Chat 多實例重複處理）。
- `Topic` 與 `Channel` 命名是否並存。
