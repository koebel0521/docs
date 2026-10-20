# TCP 封包詳細參考

本頁依現行程式整理 TCP runtime 的封包格式、介面、連線生命週期與診斷方式；閱讀順序見 [TCP 元件索引](TCP元件索引.md)，frame 位元組細節另見 [framecodec 封包格式](framecodec封包格式.md)，登入呼叫鏈見[連線與登入流程](../10-核心流程/連線與登入流程.md)，S2S 驗證的部署設定見 [S2S 身份驗證](../30-部署與開發/S2S身份驗證.md)。

## 套件分工

| 套件 | 責任 |
|------|------|
| `pkg/tcpruntime` | 契約：`Msg`、`Service`、`Socket`、`ServiceEvent`、`Drainer` 與 `AddMsgHandler`；只有介面，沒有實作 |
| `pkg/framecodec` | frame 編解碼（`Coder`）與分級緩衝池 |
| `pkg/tcpsocket` | `Socket`（收送迴圈、deadline、限流、送出快速路徑）、`Sockets`（連線集合）、`OwnerBase`（訊息註冊、廣播佇列、生命週期） |
| `pkg/tcpserver` | `Server`：監聽、accept、接入裁決（`(*Server).admitConn`）、S2S accept 握手 |
| `pkg/tcpclient` | `Client`：dial、重連退避、位址輪替、S2S dial 握手 |
| `pkg/tcpconfig` | `Config`、`ServerOptions`、listen／dial 位址解析（`BuildConfig`、`ListenAddr`、`DialAddrs`） |
| `pkg/tcpregistry` | `Registry`：登記各服務 owner、依名稱查找、`Drain` 滾動部署 |
| `pkg/servicefinder` | `Finder` 介面（`Gate()`、`Central()`、`World()`、`User()`、`Chat()`、`Lobby()`），由 `tcpregistry.Registry` 實作 |
| `pkg/s2sauth` | 服務間 TLS 1.3 + 節點 token 驗證、來源 CIDR 限制 |
| `pkg/msg`、`pkg/msg/<群組>` | 執行期訊息型別（物件池、`PackTo`／`Unpack`） |
| `pkg/msgpool` | `MsgBase[T]`、`Pool[T]` |
| `pkg/protocol`、`pkg/protocol/wire` | 由 PJA-Protocol 產生的訊息 ID、描述表與 Luban 結構，不得手改 |
| `pkg/tcpdiag` | 協定收送紀錄（`/diag/protocol` 的資料來源） |

## 封包格式

每個 frame：

```
[4B size uint32 LE][2B msgID uint16 LE][payload]
```

- `size` 含 6 byte header（`framecodec.FrameHeaderSize = 6`），小於 6 回 `ErrMsgSizeTooSmall`，超過上限回 `ErrMsgSizeExceedsLimit`，連線關閉
- 上限：`Coder.MaxFrameSize`，0 或超過 4MB 時以 4MB 計。Gate 玩家連線設 `gateUserMaxFrameSize = 64 * 1024`（`tcpserver.WithMaxFrameSize`）；服務間連線為 4MB
- payload 是 Luban 編碼的協定結構（`wire.*` 的 `AppendTo`／`DecodeFrom`）；frame 本身沒有版本或壓縮欄位
- S2S 驗證啟用時，frame 走在 TLS 之內；TLS 前的身份 frame 格式見下方 s2sauth

### 編碼

- `Coder.WriteFrame(w, id, payload)`：依 frame 大小從分級池取緩衝（超過最大等級則直接配置），複製 payload 後一次 `Write`，沒有堆疊陣列的小訊息路徑
- `Coder.WriteFramePrebuilt(w, id, frame)`：呼叫端已在 frame 前預留 6 byte，只填 header 後寫出。`Socket` 的送出快速路徑用它（見下方 PackTo）

### 解碼

- `Coder.DecodePooled(r)` 回傳 `(id, payload, release, err)`；payload 借自池，處理完必須呼叫 `release`
- 分級池：8KB／64KB／512KB；超過 512KB 的緩衝直接配置、不還池
- 大於 64KB 的 frame 依實際收到的資料逐步擴充（初始 64KB、每次加倍），不依對端宣告的長度預先配置，避免只送 header 的連線佔住 4MB
- `Coder.Decode(r, b)` 是使用呼叫端暫存的非池版本

## 訊息 ID

`protocol.MessageID` 是 `struct{ Value uint16 }`（`pkg/protocol/types`）；數值定義在 `wire.ProtocolMessageID_*`，`protocol.LookupMessage`（Client 訊息）提供名稱、種類（`MessageKindRequest`／`Response`／`Command`／`Event`）與方向。`protocol.ProtocolVersion`、`protocol.SchemaHash` 標示產生版本。

| 範圍 | 用途 | 例 |
|------|------|------|
| 0 | `ProtocolMessageID_Unknown`；玩家連線收到 0 記 Warn 後直接斷線，S2S 連線記 Error 後略過該 frame、保持連線（`handleInvalidMsgID`） | |
| 1–31 | Client 盤面、戰鬥、任務 | `LoadPlayerRequest = 1`、`RollDiceCommand = 3`、`ChangeSetEvent = 31` |
| 1001–1004 | Client 登入登出 | `LoginRequest = 1001`、`LogoutResponse = 1004` |
| 2005–2011 | Client 連線：心跳、校時、版本握手 | `HeartbeatRequest = 2005`、`HelloRequest = 2009`、`VersionNotice = 2011` |
| 2201–2206 | Client 聊天 | `ChatJoin = 2201`、`ChatSync = 2206` |
| 30001–30013 | S2S 登入登出鏈 | `LoginGateToCentral = 30001`、`LoginRejectedGateToCentral = 30012`、`LogoutBatchCentralToWorld = 30013` |
| 30101–30105 | S2S 連線 | `ConnectInfo = 30101`、`HeartbeatReq = 30102`、`HeartbeatRes = 30103`、`TimeSync = 30104`、`ServiceHello = 30105` |
| 30201–30202 | S2S 聊天 | `ChatSystem`、`ChatDeliverBatch = 30202` |
| 30301–30304 | S2S 轉發 | `ForwardToWorld`、`SyncUsersGateToCentral`、`ClientBoardCommand`、`BoardClientEvent` |
| 30401–30402 | S2S 對帳 | `AuditSnapshotRequest`、`AuditSnapshotResponse` |
| 30403–30405 | 頻道（已停用） | `ChannelBroadcast`、`ChannelMembershipUpdate`、`ChannelMembershipSnapshot`；Go 端已無發送與接收，定義待下次升版移除 |
| 30501–30502 | S2S 帳號保留 | `AccountRetentionRequest`、`AccountRetentionResponse` |

完整清單以 `pkg/protocol/wire/Protocol.ProtocolMessageId.go` 為準。

## 訊息型別慣例

`tcpruntime.Msg` 只要求四個方法：

```go
type Msg interface {
	MsgID() protocol.MessageID
	Pack() []byte
	Unpack([]byte) bool
	Put()
}
```

生成的訊息（例如 `pkg/msg/auth/login_response.go`、`pkg/msg/system/service_hello.go`）都長這樣：

- 結構嵌入 `msgpool.MsgBase[T]` 與對應的 `wire.*` 結構；`MsgBase` 提供 `MsgID()` 與 `Put()`（歸還 `Pool`）
- `NewXxx()` 從池取物件並 `Init(id, pool, self)`；`Reset()` 清欄位
- `PackTo(buf []byte) []byte` 把編碼接在 `buf` 之後；`Pack()` 委託 `PackTo(nil)`
- `Unpack(data) bool` 解碼失敗回 false，訊息保持原值
- 帶追蹤欄位的訊息實作 `msg.RequestScopedMessage`（`RequestMeta()`）或 `msg.TraceScopedMessage`（`TraceMeta()`）；`msg.NextRequestID`、`msg.NextTraceID` 產生行程內單調 ID

所有權規則：

- `SendMsg` 之後訊息歸 socket，送完由 sendLoop `Put`；呼叫端不可再使用
- 收到的訊息由 handler 事件執行完後 `Put`；handler 要跨 goroutine 保留資料須先複製
- `msg.OwnedRaw`（`TakeOwnedRaw`）只送單一 socket，交換緩衝不複製，送出後歸還池；原本可多 socket 共用的 `msg.Raw` 已無呼叫端，已移除
- `system.SharedHeartbeatReq` 是不回收的單例（pool 為 nil），心跳廣播重複送它

### PackTo 快速路徑

`Socket.handleOutgoing` 對實作 `PackTo` 的訊息：在 socket 自有的 `sendBuf` 前預留 6 byte，呼叫 `PackTo` 後用 `WriteFramePrebuilt` 寫出，省掉 `Pack()` 的配置與一次 payload 複製。`sendBuf` 只由 sendLoop 使用，容量超過 `maxRetainedSendBuf`（64KB）時不保留。沒有 `PackTo` 的訊息走 `Pack()` + `WriteFrame`。

### 預序列化訊息（writev 路徑）

已經是 wire bytes 的訊息（`OwnedRaw`、`packedMsg`，實作套件內介面 `prepackedMsg`，即 `Payload() []byte`）不再複製進 `sendBuf`，改走 `writePrepacked`：

- payload 放得進 bufio 剩餘空間：`Write` 6 byte header 與 payload，只剩 bufio 一次複製
- 放不下：先 `Flush`，再以 `net.Buffers{hdr, payload}.WriteTo(conn)` 一次 writev 寫出，零複製
- header 與 iov 陣列放在 `Socket` 結構內，不逃逸，送出路徑零配置
- 超過 frame 上限或短寫回傳錯誤，連線關閉

## 介面

### Service

```go
type Service interface {
	socketFinder   // Get(key)、ByID(id)、All()、Random()、Range(fn)、Len()
	Broadcaster    // SendAll(Msg)
	AppID() int64
	Name() string
	Close()
	MsgFactory(protocol.MessageID) MsgFactory
	MsgHandler(protocol.MessageID) MsgHandler
	AddNewMsg(func() Msg, func(Service, Socket, Msg))
	Async(func())
}
```

`tcpserver.Server` 與 `tcpclient.Client` 都嵌入 `*tcpsocket.OwnerBase` 實作它；`Server` 另實作 `Drainer`（`StopAccepting()`）。只需部分能力的消費端收 `socketFinder`、`Broadcaster` 或自訂小介面。

### Socket

```go
type Socket interface {
	Peer                 // Key()、ID()、SendMsg(Msg)、Context()
	Data() any
	RemoteAddr() string
	LocalAddr() string
	PushEvent(func()) bool
	Start()
	Close()
	Wait()
	SetKey(string)
	SetID(int64)
	SetData(any)
}
```

- `Key()`：建立時由 `netx.NewSocketKey()` 產生的連線唯一鍵；`ID()`：綁定的玩家 ID 或節點 ID（`Sockets.ByID` 依此索引）
- `Context()`：handler 執行期間帶該訊息 traceID 的 context；連線結束時取消
- `PushEvent(fn)`：把任意工作送進事件迴圈，回傳 false 代表沒送入、fn 不會執行；現在經 host 投遞，與 `AppBase.PostEvent` 同保證：事件迴圈結束後一律回 false

### ServiceEvent

```go
type ServiceEvent interface {
	Connect(Service, Socket)
	Disconnect(Service, Socket)
	Init(Service)
}
```

各服務 handler 實作它：`Init` 註冊訊息，`Connect`／`Disconnect` 處理連線事件。`s2shandshake.ConnectInfoOnConnect` 可嵌入作為 S2S 的 `Connect` 預設實作。

### 註冊 handler

```go
tcpruntime.AddMsgHandler(svc, system.NewHelloRequest, r.onHelloRequest)
```

`AddMsgHandler[T]` 包裝型別斷言後呼叫 `OwnerBase.AddNewMsg`。同一 ID 重複註冊只保留第一個並記錄錯誤。owner 為 S2S owner（已驗證，或建構時沒有標 `WithUntrustedPeer`；與是否開啟 `s2s_auth` 無關，開發環境與正式環境語意一致，只有面向玩家的 owner 例外）時，handler 在註冊時包上 `guardSerialized`：連線 context 已取消或該 socket 已不在 owner 的連線集合中（例如被同節點新連線取代）時，訊息直接略過。

## 建構

### Server

```go
svc := tcpserver.New(name, baseApp, ownerEvent,
	tcpserver.WithConfig(tcpCfg),
	tcpserver.WithWriteDeadTime(tcpsocket.PlayerWriteDeadline),
	tcpserver.WithMaxFrameSize(gateUserMaxFrameSize),
	tcpserver.WithSocketOptions(
		tcpsocket.WithRateLimit(rate.Limit(gateUserRatePerSec), gateUserRateBurst),
		tcpsocket.WithUntrustedPeer(),
		tcpsocket.WithSendQueueDepth(gateUserSendQueueDepth),
	),
)
registry.Register(svc)
go svc.Start()
```

（節錄自 `cmd/gate/internal/services/userserver.go`。）選項：

| 選項 | 作用 |
|------|------|
| `WithConfig(tcpCfg)` | 讀寫逾時設為 `tcpsocket.ServerDeadline`，並套用 `tcpCfg` 的 `ServerOptions` 與位址解析 |
| `WithReadDeadTime`／`WithWriteDeadTime` | 只接受正值；未設定預設 30 秒 |
| `withServerOptions(tcpconfig.ServerOptions)` | 來源 IP 限流、`MaxConns`、`SocketSendQueueDepth` |
| `WithTCPConfig(cfg)` | 只設定位址解析與握手 |
| `WithSocketOptions(...)` | socket 層選項（型別為 `tcpsocket.Option`） |
| `WithReadyRegister(fn)` | 就緒檢查的註冊函式；正式環境傳 `debug.Server` 的 `RegisterReadyChecker`（由 `startup.InitializeObservability` 回傳的 `*debug.Server` 經組合根傳遞），不設定時不註冊。全域的 `debug.Default`、`debug.RegisterReadyChecker`、`debug.CheckReady` 已刪除 |
| `WithMaxFrameSize(n)` | 收包 frame 上限 |

socket 選項：`WithRateLimit`（每條連線獨立的讀取限流，超過即關閉）、`WithUntrustedPeer`（玩家連線，見事件派發）、`WithSendQueueDepth`（送出佇列深度，Start 前套用）、`withEventCh`（`NewOwnerBase` 自動附加）。

### Client

```go
svc := tcpclient.New(name, baseApp, ownerEvent,
	tcpclient.WithTCPConfig(tcpCfg),
	tcpclient.WithReadDeadTime(...), tcpclient.WithWriteDeadTime(...))
```

### 設定與位址

`tcpconfig.BuildConfig(v, appID)` 每次解析都即時讀 viper：

- listen：`<appID>.listen.<service>.ip`（`host:port`），`ParseListenAddr` 解析為 IPv4 TCP 位址
- dial：`<appID>.dial.<service>` 底下每個項目的 `ip`（`host:port` 字串），`ParseDialAddrs` 只檢查格式與埠號後排序去重，不做 DNS 解析；部分項目無效時略過，全部無效才回錯
- `tcp.rate_limit_per_second`（預設 10）、`tcp.rate_limit_burst`（預設 20）、根層 `max_connections`（正式鍵，0 不限制；`tcp.max_connections` 為已棄用別名，有設定時仍優先並記 Warn）、`tcp.socket_send_queue_depth`（預設 `tcpconfig.DefaultSocketSendQueueDepth = 10000`）；只有大於 0 的值才覆蓋預設（`ParseServerOptions`）

## 連線生命週期

### Server 端

1. `Server.Start()` → `OwnerBase.RunLifecycle(prepareListen, runServer)`：`prepareListen` 以 `tcpconfig.ListenAddr` 解析位址；成功後建立 `Sockets`，`runServiceBody` 呼叫 `ServiceEvent.Init`、啟動廣播佇列 goroutine、再在背景跑 `runServer`，`Start` 阻塞到 owner 結束
2. `listen()` 監聽成功或失敗後註冊就緒檢查 `tcp_listen_<name>`（經 `WithReadyRegister` 傳入的 `RegisterReadyChecker`）：監聽失敗、尚未監聽或已 `StopAccepting` 時 `/ready` 回 503；監聽失敗時行程不退出
3. `acceptLoop()`：
   - `Accept` 回 `net.ErrClosed` 或 owner 已關閉時結束；其他錯誤（EMFILE、ECONNABORTED 等）以 5ms 起加倍、上限 1 秒退避後重試
   - `admitConn`：先檢查來源 IP 限流（`ratelimit.IPLimiter`；表滿時淘汰最久未見的來源，不拒絕新來源），再取 `connSem` 槽位（`MaxConns`）；被拒直接關閉
   - 並發上限 `connSem`（`MaxConns`，未設定為 10000）滿時直接拒絕，不阻塞 accept loop
4. `handleConn`：有 `AcceptHandshake[name]` 時先做 S2S 驗證，取得已驗證的節點 ID；失敗記 Warn 後關閉
5. `OwnerBase.ServeConn(conn, id)`：建 socket、`id != 0` 時先 `SetID`、加入 `Sockets`、`Start()`（真實 `Socket` 只啟動送出迴圈；測試替身沿用收送兩個 goroutine）、`Connect` 回呼、由 `ServeConn` 自己的 goroutine 直接跑接收迴圈（每條連線少一個只在 `Wait` 阻塞的 goroutine，約 2 個／連線，原為 3 個）再 `Wait()` 送出迴圈結束、`Disconnect` 回呼並從 `Sockets` 移除
6. `Close()`：`CloseRuntime` 標記關閉、關閉所有 socket 與 listener，等待所有 goroutine 結束

### Client 端

1. `Client.Start()` → `RunLifecycle(prepareDial, runClient)`：`prepareDial` 以 `tcpconfig.DialAddrs` 取得已排序的設定位址（`host:port` 字串）；主機名在每次撥號時才由 Dialer 解析，啟動時對端 DNS 還解析不到也不影響
2. `dialLoop` 先等 1 秒，接著 `dialWithRetry`：預設是**備援模式**，同一時間只有一條 dial goroutine、一條連線，位址清單依序嘗試；指定 `WithDialAll()` 且位址多於一個時改為**全連模式**：`runClient` 為每個位址啟動一個獨立的 `dialTargetLoop`，各自維持一條連線、各自退避重連（Gate 連 World 使用全連模式，見[系統導讀](../00-導讀/系統導讀.md#gate-與-world-的連線)）
3. 每次 dial：`tcp_reconnect_attempts_total` +1；有 `DialHandshake[name]` 時做 S2S 握手
4. 失敗：`tcp_reconnect_total{result="fail"}` +1，依退避等待後換下一個位址（round-robin）；備援模式整輪失敗後重讀設定清單；全連模式每個迴圈綁定一個設定項目，只在自己的位址重試（對端換 IP 或暫時解析不到都不會改撥別台）
5. 成功：`tcp_reconnect_total{result="success"}` +1，退避重設，`ServeConn(conn, 0)` 阻塞到斷線；斷線後等一次退避間隔（重設後約 1 秒）再連同一位址
6. `Close()` 後迴圈結束，不再重連

### 重連退避

`tcpclient.defaultReconnectBackOff()`：`cenkalti/backoff/v5` 的 `ExponentialBackOff`，初始 1 秒、倍率 2、上限 30 秒、jitter 0.3；成功連上即 `Reset()`。`nextDelay` 至少 1ms，`backoff.Stop` 時以 30 秒計。等待用 `OwnerBase.SleepOrStop`，owner 關閉時立即返回。

### 關閉 socket

- `Socket.Close()` 只執行一次：標記關閉、送 nil 停止訊號給 sendLoop，sendLoop flush 已排隊的訊息後關閉連線並排空剩餘訊息（歸還池）
- 送出佇列已滿放不進停止訊號時直接關閉 `net.Conn`，並在背景補送停止訊號，避免 `Wait()` 卡住
- receiveLoop 結束（EOF、解碼錯誤、限流、玩家協定違規、讀取逾時）一律 `Close()`

## 事件派發

所有連線收到的訊息都送進該 app 唯一的事件迴圈（`AppHost.EventCh()`，元素為 `appcore.Event`），handler 在事件迴圈執行，不需自行加鎖。

收包流程（`Socket.receiveOnce`／`handleIncoming`）：

1. 節流設定 read deadline 後 `DecodePooled`
2. 讀取限流超過 → 關閉
3. msgID 為 0 → 玩家連線記 Warn 後關閉連線（無合法用途，避免每則灌一行日誌且繞過未 Hello 檢查）；S2S 連線記 Error 後略過、連線保留
4. 用 owner 的 factory 建訊息、`Unpack`、歸還緩衝
5. 記錄協定事件（`tcpdiag.RecordEvent`）、`tcp_messages_received_total` +1
6. `pushMsgEvent`：從池取 `msgEvent` 送進 eventCh；事件執行時若訊息實作 `TraceScopedMessage`，先把 traceID 注入 `Socket.Context()` 再呼叫 handler，結束後 `Put` 訊息

玩家連線（`WithUntrustedPeer`）與服務間連線的差異：

| 情況 | 玩家連線 | S2S 連線 |
|------|----------|----------|
| 未註冊的訊息 ID 或沒有 handler | 關閉連線 | 記錄並丟棄（`tcp_unknown_messages_total`） |
| `Unpack` 失敗 | 關閉連線 | 丟棄（`tcp_dropped_messages_total{reason="unpack"}`），連線保留 |
| handler panic | 關閉連線 | 丟棄（`reason="handler_panic"`），連線保留 |
| eventCh 已滿 | 關閉連線 | 阻塞等待（背壓）直到送入或連線結束 |
| 解碼錯誤日誌 | Warn | Error |

eventCh 滿時兩者都計 `game_event_queue_overflow_total`。

### 送出與廣播

- `Socket.SendMsg`：非阻塞送入 `sendCh`；佇列已達深度（`WithSendQueueDepth` 或 `DefaultSocketSendQueueDepth`）代表對端長時間不讀，直接關閉連線並計 `tcp_send_queue_overflow_total`。Gate 玩家連線深度 1024
- sendLoop 單一 goroutine 依序寫入 `bufio.Writer`（4KB），佇列空了才 `Flush`，連續訊息合併成一次寫入
- `OwnerBase.SendAll(msg)`：直接呼叫非阻塞的 `Sockets.SendAll`，沒有 owner 佇列與轉送 goroutine（舊的 owner 廣播佇列已移除）；owner 已關閉則歸還 msg 並略過。`Sockets.SendAll` 只 `Pack` 一次，包成引用計數的 `packedMsg` 分送給快照中的每個 socket
- `OwnerBase.Async(f)`：在 owner 的 WaitGroup 下開 goroutine，`Close` 會等它結束

## Deadline

| 常數 | 值 | 用途 |
|------|----|------|
| `tcpsocket.ServerDeadline` | `3 * appcore.HeartbeatInterval`（180 秒） | `tcpserver.WithConfig` 的服務間連線與 Gate 玩家連線讀寫逾時 |
| `tcpsocket.PlayerWriteDeadline` | 30 秒 | Gate 寫給玩家連線的逾時 |
| `tcpsocket.UserDeadline` | 2 分鐘 | user 服務（模擬玩家）連往 Gate 的讀寫逾時 |
| `tcpserver` 預設 | 30 秒 | 未指定讀寫逾時的 server |
| `tcpclient` 的 `defaultDialTimeout` | 10 秒 | 預設 `dial` 函式（`net.Dialer`） |
| `s2sauth` 握手 | 5 秒 | TLS 與身份 frame 完成前的整體 deadline，完成後清除 |

讀寫 deadline 都有節流：距上次設定未超過逾時的 1/4 時不重設，避免每則訊息一次 `time.Now()`。逾時 0 代表不設 deadline。

## 心跳

- 服務間：`appcore.HeartbeatInterval = 60s`。Gate 連 Central、World、Chat 的 client owner 以 `s2shandshake.BroadcastHeartbeat` 每 60 秒廣播 `SharedHeartbeatReq`；對端以 `system.HandleHeartbeatReq`（`pkg/msg/system`）回 `HeartbeatRes`，發起端用 `s2shandshake.IgnoreHeartbeatRes` 忽略回覆。心跳只用來維持讀取活性：3 個間隔（`ServerDeadline`）收不到任何資料就讀取逾時斷線
- 玩家：Client 送 `HeartbeatRequest`（2005），Gate `onHeartbeatReq` 回 `HeartbeatResponse`；已登入的連線才刷新最後心跳時間。Gate 每 5 秒（`userConnCheckInterval`）掃描，超過 `userHeartbeatTimeout = 90s` 沒心跳即斷線（逾時偵測上限約 95 秒）（`game_heartbeat_timeout_total{service="gate_user"}`）；建立連線後 30 秒（`userLoginHandshakeTimeout`）內未完成登入也斷線

## ConnectInfo 與 ServiceHello

S2S 連線建立後，`s2shandshake.ConnectInfoOnConnect.Connect` 依序送出：

1. `ConnectInfo`（30101）：本節點 `AppID`。對端以 `system.BindConnectInfo`（`pkg/msg/system`；只綁定的處理器用 `s2shandshake.BindConnectInfoOnly`）綁到 socket ID；socket 已綁其他 ID 時關閉連線
2. `ServiceHello`（30105）：`s2shandshake`（套件內 `sendServiceHello`）送出 `ServiceIdentity`（`Release`、`ServerCommit`、`ProtocolSchemaHash`、`ClientContract`，另帶只供診斷的 `AppType`＝`ServiceKind` 與 `AppID`）

接收端以 `s2shandshake.RegisterServiceHello(svc)` 註冊處理：`verifyServiceHello` 任一項與本節點不同就記錄 `service hello mismatch, closing socket`、計 `game_service_hello_mismatch_total` 並關閉連線，擋下混版本部署。身分由啟動流程 `s2shandshake.SetServiceIdentity` 設定；沒設定（單元測試）時不送、不驗證也不要求。

設定身分後 hello 為強制：`ConnectInfoOnConnect.Connect` 送出自己的 hello 後登記等待對端 hello，`s2shandshake.ServiceHelloTimeout`（5 秒）內沒收到就記錄 `service hello timeout, closing socket` 並關閉；`s2shandshake.RegisterServiceHello` 同時對支援 `tcpruntime.MsgGuardSetter` 的服務（`OwnerBase.SetMsgGuard`，守門掛在各服務實例上，不是行程全域）裝上前置檢查，等待中的連線收到 `ConnectInfo`、`ServiceHello` 以外的訊息時記錄 `message before service hello, closing socket` 並關閉。所有 S2S 方向（Central-Gate、Central-World、Central 與 Chat／Lobby 的時間同步、Gate-World、Gate-Chat、World-Central、World-Gate）都經 `ConnectInfoOnConnect` 收送 hello。

## s2sauth

`s2s_auth.enabled` 必須明確設定（漏設視為設定錯誤）；`false` 時服務間為明文，每個行程警告一次。

- 監聽端：`s2sauth.ConfigureServerFor(v, cfg, service, prefix)` 讀 `<prefix>.cert_file`、`key_file`（每 30 秒重讀憑證）、`tokens_file`（節點 ID → `sha256:<hex>`，檔案變更時 `updateTokens` 原子替換並關閉被撤銷的連線）、`allowed_cidrs`，安裝 `cfg.AcceptHandshake[service]`
- 連出端：`s2sauth.ConfigureClientFor(v, cfg, nodeID, service, prefix)` 讀 `<prefix>.ca_file`、`token_file`、`server_name`，安裝 `cfg.DialHandshake[service]`；每次 dial 重讀檔案

握手（5 秒內完成）：

1. 監聽端先比對來源 IP 是否在 `allowed_cidrs`
2. TLS 1.3 握手，連出端驗證伺服器憑證與 `server_name`（不允許 `InsecureSkipVerify`）
3. 連出端送身份 frame：`"PJA1"` + 節點 ID（int64 BE）+ token 長度（uint16 BE，43–256）+ token（`s2sauth.IdentityFrame`）
4. 監聽端以固定時間比對 token 雜湊，通過回 1 byte ack，失敗直接斷線（連出端 `ReadAck` 回錯）
5. 同一節點以有效憑證重連時取代並關閉舊連線（多半是對端重啟留下的半開連線）

驗證通過的節點 ID 交給 `ServeConn` 預先 `SetID`，且 S2S owner 啟用 `guardSerialized`（斷線與執行中的訊息互斥，且只處理仍在 owner 連線索引內的連線的訊息；與 `s2s_auth.enabled` 無關，關閉驗證的開發環境與整合測試走同一套語意，例如 Chat 斷線後晚到的 Join 不會再把成員掛回已斷的 Gate）。

## 滾動部署

`tcpregistry.Registry.Drain(ctx, timeout, interval)` 對實作 `Drainer` 的 owner 呼叫 `StopAccepting()`（關 listener、`/ready` 轉為失敗、既有連線保留），等待連線自然結束；回傳逾時時仍存活的 socket 數。主動連出的 client 不列入等待。`DrainService` 只排空單一 owner。

## 診斷

### 指標

| 指標 | 說明 |
|------|------|
| `tcp_connections_active`、`tcp_connections_total` | 依 `service`（owner 名稱）計 |
| `tcp_messages_sent_total`、`tcp_messages_received_total` | 收送訊息數 |
| `tcp_send_queue_depth`、`tcp_send_queue_overflow_total` | socket 送出佇列深度（scrape 時加總各 socket 的 `len(sendCh)`，不再逐則 Inc/Dec）與溢位關閉次數 |
| `tcp_unknown_messages_total` | 未註冊訊息 |
| `tcp_dropped_messages_total{reason}` | S2S 的 `unpack`／`handler_panic` 丟棄，以及 `loop_closed`（事件迴圈已停止後才收到的 S2S 訊息，不執行、歸還物件池並計數）；World 盤面指令另有 `board_queue_full`（工作者佇列滿）與 `unknown_player`（玩家不在本 World 或所屬 Gate 不符，同時有取樣 Warn）、`board_decode`（盤面／任務指令 payload 解碼失敗，以 `board.ErrDecode` 區分，記取樣 Warn 而非 Error） |
| `tcp_reconnect_attempts_total`、`tcp_reconnect_total{result}` | client dial 次數與成敗；`tcp_reconnect_success_percent_5m` 為錄製規則 |
| `game_event_queue_overflow_total` | eventCh 滿 |
| `game_msg_queue_dropped_total` | 目前只剩 Chat 房間工作佇列滿（`service="chat_room"`）；owner 廣播佇列已移除 |
| `game_service_hello_mismatch_total` | 發版身分不一致、逾時未送 hello 或 hello 前送其他訊息 |
| `game_heartbeat_timeout_total` | 玩家心跳逾時 |

### 協定紀錄

`tcpdiag` 在收送時記錄 `ProtocolRecord`（時間、owner、方向 `recv`／`send`、狀態、msgID、requestID、traceID、socket key／ID、位址、錯誤，選擇性含 payload），由 debug HTTP 的 `/diag/protocol`（`debug.ProtocolDiagPath`）查詢，可依 traceID／requestID 過濾（`tcpdiag.FilterRecords`）。

開關（`pkg/startup/runtime.go`，熱重載後仍有效）：

- 啟用：旗標 `-protocol-record` > 環境變數 `DEBUG_PROTOCOL_RECORD` > 設定 `protocol_record_enabled`
- payload：`protocol_record_payload_enabled` 允許時，再由 `DEBUG_PROTOCOL_RECORD` 決定
- 容量與保留：`protocol_record_limit`、`protocol_record_ttl`；落檔目錄 `protocol_record_file_dir`（目錄權限 0700、檔案 0600，內含 socket key 與訊息內容；開檔／寫入失敗只在由成功轉失敗時記一筆日誌，避免磁碟滿時洗版）

### 日誌

- `socket close`（Info；debug 等級另含呼叫鏈 `callers`）、`connect`／`disconnect`（Info，含 key 與位址）
- `send queue overflow, closing socket`、`event queue overflow, closing socket`、`rate limit exceeded, closing connection`、`unknown message dropped`（Warn）
- `S2S authentication rejected`、`S2S node reconnected, closing its previous connection`（Warn）、`service hello mismatch, closing socket`、`service hello timeout, closing socket`、`message before service hello, closing socket`（Error）
- `dialing`、`dial error`（client）、`accept error, retrying`（server）
