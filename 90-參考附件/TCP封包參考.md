# TCP 封包詳細參考

本文件保留完整結構、介面與函式細節；閱讀入口請先看[TCP 元件索引](TCP元件索引.md)。

這份文件整理 `pkg/tcpserver`、`pkg/tcpclient`、`pkg/tcpsocket`、`pkg/msgpool` 與相關 TCP 輔助套件的主要結構、介面與函式用途，目標是幫你快速建立「整個 TCP runtime 怎麼運作」的心智模型，而不是只看零散 API。

`Server` / `Client` / `OwnerBase` / `ServerOptions` / transport / socket / coder / 訊息池等 runtime orchestration 已移到 [`pkg/tcpserver`](../../pkg/tcpserver/server_listen.go)、[`pkg/tcpclient`](../../pkg/tcpclient/client.go)、[`pkg/tcpsocket`](../../pkg/tcpsocket/socket.go) 與 [`pkg/msgpool`](../../pkg/msgpool/pool.go)，對外最小介面在 [`pkg/tcpruntime`](../../pkg/tcpruntime/tcpapi_service.go)（原 `pkg/tcpapi`），服務查找介面在 [`pkg/servicefinder`](../../pkg/servicefinder/servicefinder.go)，協定記錄 core 在 [`pkg/tcpdiag`](../../pkg/tcpdiag/protocol_record.go)，`/diag/protocol` 由 [`pkg/debug`](../../pkg/debug/server.go) 掛載，重連退避策略已整併至 [`pkg/tcpclient`](../../pkg/tcpclient/client_reconnect.go)，Registry 在 [`pkg/tcpregistry`](../../pkg/tcpregistry/registry.go)，而 `registry.NewService(...)` 會直接接收 `serverOpts`、client 位址查詢函式與可選的 `socketOpts`。

## 整體定位

TCP runtime 相關套件負責：

- 依照設定建立 server/client 連線
- 管理多個 socket
- 封包編解碼
- 訊息 factory / handler 註冊與派發
- 連線生命週期與重連
- 廣播、單點傳送、依 ID 查找 socket

它的分層大致如下：

1. `Server` / `Client` 代表一種服務連線角色
2. `serviceBase` 提供共用能力：socket 建立、send queue、msg registry
3. `Socket` 代表單一 TCP 連線
4. `Sockets` 管理某個 service 下面的所有連線
5. `Coder` 與 `packedMsg` 處理訊息編碼與廣播封裝

最小介面在 [`pkg/tcpruntime`](../../pkg/tcpruntime/tcpapi_service.go)（原 `pkg/tcpapi`），`Server` 的 runtime orchestration 在 [`pkg/tcpserver`](../../pkg/tcpserver/server_listen.go)，`Client` 的 runtime orchestration 在 [`pkg/tcpclient`](../../pkg/tcpclient/client.go)，`Socket` / `Sockets` / `Coder` 實作在 [`pkg/tcpsocket`](../../pkg/tcpsocket/socket.go)，服務查找介面在 [`pkg/servicefinder`](../../pkg/servicefinder/servicefinder.go)，協定記錄 core 在 [`pkg/tcpdiag`](../../pkg/tcpdiag/protocol_record.go)，`/diag/protocol` 路由在 [`pkg/debug`](../../pkg/debug/server.go)，Registry 在 [`pkg/tcpregistry`](../../pkg/tcpregistry/registry.go)。

## 執行流程

### Server 端

1. `tcpregistry.NewService(...)` 建出 `Server`
2. `Server.Start()` 依設定讀出 listen address
3. `Server.listen()` 啟動 listener
4. `acceptLoop()` 接受新連線
5. 每條連線包成一個 `Socket`
6. `Socket.receiveLoop()` 收包、解碼、派發 handler
7. `Socket.sendLoop()` 處理送包

### Client 端

1. `tcpregistry.NewService(...)` 建出 `Client`
2. `Client.Start()` 依設定讀出 dial address 列表並正規化順序
3. 只維持一條 `dialLoop()`
4. 連上後建立 `Socket`
5. socket 中斷時依 `ReconnectStrategy` 重連，失敗時切到下一個候選位址
6. `Close()` 後會停止後續重連

## `tcpserver`

`tcpserver` 是 TCP server runtime orchestration 的實作層。

### 主要職責

- 解析 `ServerOptions`、listen 位址
- 建立 server runtime
- 管理 accept 連線與 socket 生命週期

## `tcpclient`

`tcpclient` 是 TCP client runtime orchestration 的實作層。

### 主要職責

- 解析 dial 位址
- 建立 client runtime
- 管理 socket 生命週期與重連（含 reconnect/backoff）
- 提供 rate limit 抽象

### 主要檔案（tcpserver）

| 檔案 | 主要責任 |
|------|----------|
| `server.go` | 監聽、accept、server 生命週期 |
| `server_test.go` | runtime server 行為測試 |
| `config.go` | 從 viper 解析 listen address 與 `ServerOptions` |
| `transport.go` | 抽象化 listen，方便測試替換 |

### 主要檔案（tcpclient）

| 檔案 | 主要責任 |
|------|----------|
| `client.go` | 主動撥號、重連、client 生命週期 |
| `client_test.go` | runtime client 行為測試 |
| `ratelimit.go` | IP token bucket 限流器 |
| `config.go` | 從 viper 解析 dial address |
| `transport.go` | 抽象化 dial，方便測試替換 |

## `tcpsocket`

### 介面與結構

- `type Socket struct`
  單一 TCP 連線的收發與訊息派發。

- `type Sockets struct`
  多 socket 容器與索引。

- `type Coder struct`
  TCP frame 編解碼器。

### 函式

- `NewSocket(...)`
  建立 socket。

- `NewSockets()`
  建立 sockets 容器。

- `newSocket(...)`
  內部 socket 建構 helper，供測試使用。

- `newSockets()`
  內部容器建構 helper，供測試使用。

## `service.go`

### 常數

- `DeadlineServer`
  用於服務器之間連線的預設讀寫 timeout。

- `DeadlineUser`
  用於 `gate <-> user` 這種面向玩家連線的預設讀寫 timeout。

### 介面與結構

- `type IServiceEvent interface`
  service 生命週期 callback。
  包含：
  - `Connect`
  - `Disconnect`
  - `Init`
  - `Destroy`

- `type IService interface`
  給上層 handler 使用的統一 service 能力。
  重要方法：
  - 查詢 socket：`Get`、`GetByID`、`GetRandom`
  - 發送訊息：`Send`、`SendByID`、`SendAll`
  - 註冊訊息：`AddNewMsg`
  - 取得訊息 factory / handler：`GetMsgFactory`、`GetMsgHandler`
  - 基礎設定：`SetCoder`、`SetTransport`
  - 背景執行：`Async`

## `service_base.go`

`serviceBase` 是 `Server` 與 `Client` 的共用底層。

### 結構

- `type serviceBase struct`
  主要欄位：
  - `baseApp`
    對應 app runtime，用來拿 `GetID()`、`GetContext()`、事件 channel。
  - `sockets`
    這個 service 名下的所有連線。
  - `dispatcher`
    收到訊息後把 handler 丟去哪裡跑。
  - `coder`
    封包編解碼器。
  - `transport`
    真正做 `Listen` / `Dial` 的抽象。
  - `sendCh`
    service 層級的送包 queue。
  - `msgFactories` / `msgHandlers`
    以 `MsgID` 為索引的訊息工廠與 handler 表。

- `type socketFactory`
  socket 建立函式型別，方便測試注入假 socket。

### 函式

- `newServiceBase(...)`
  初始化共用基底。

- `initSockets()`
  建立 `Sockets` 容器。

- `GetAppID()`, `GetName()`
  取 app ID / service name。

- `markClosed()`, `stop()`
  標記 service 關閉並停止背景 queue。

- `SetSocketFactory(...)`
  替換 socket 建立方式，主要給測試。

- `SetEventDispatcher(...)`
  替換事件派發器。

- `SetCoder(...)`
  替換 message coder。

- `SetTransport(...)`
  替換 listen/dial 實作。

- `transportValue()`
  取有效 transport；沒設時回全域預設。

- `createSocket(...)`
  用目前的 `socketFactory` 建出 socket。

- `runAsync(f)`
  啟 background goroutine 並記錄 waitgroup。

- `sendAsync()`
  啟 service 層級送包 loop，從 `sendCh` 取工作執行。

- `enqueue(f)`
  把送包工作排進 queue。

- `SendAll`, `Send`, `SendByID`
  封裝成非同步工作，再交給 `Sockets` 實際送出。

- `Get`, `GetByID`, `GetRandom`
  從 `Sockets` 容器查 socket。

- `AddNewMsg(new, event)`
  把某個 `MsgID` 對應的 factory / handler 註冊進 service。
  `Socket.receiveLoop()` 解包後就是靠這裡找到 handler。

- `GetMsgFactory(id)`, `GetMsgHandler(id)`
  取回對應訊息工廠與 handler。

- `Async(f)`
  開 background goroutine。

## `server.go`

### 結構

- `type Server struct`
  `serviceBase` 之上的 server 實作，增加：
  - `listenIPInfo`
  - `allowedIPs`
  - `listener`

### 函式

- `newServer(...)`
  建立 server。

- `GetAllowedIPs()`, `SetAllowedIPs(...)`, `AddAllowedIPs(...)`
  控制允許連入的 IP 白名單。

- `isAllowedIP(ip)`
  檢查某個 IP 是否可連入。

- `Connect(sock)`
  server 側 socket 建立完成後觸發：
  - 打 log
  - 呼叫上層 `IServiceEvent.Connect`

- `Disconnect(sock)`
  server 側 socket 中斷後觸發：
  - 從 `Sockets` 移除
  - 呼叫上層 `IServiceEvent.Disconnect`

- `Init(s)`, `Destroy(s)`
  對應 service 啟動/結束時呼叫上層 lifecycle。

- `Start()`
  入口。解析 listen address、建立 socket 容器、啟 background loop。

- `start()`
  啟送包 goroutine，並另開 goroutine 跑 `runServer()`。

- `Close()`, `close()`
  關閉 server、listener、所有 socket。

- `prepareListen()`
  從設定讀出 `listen.<service>.ip`。

- `runServer()`
  server 主流程：
  - `Init`
  - `listen`
  - `acceptLoop`
  - wait 所有 goroutine
  - `Destroy`

- `listen()`
  實際呼叫 transport 開 listener。

- `startAcceptLoop()`, `acceptLoop()`
  持續 `Accept()` 新連線。

- `logAcceptErr(err)`
  accept 出錯時記錄 log。

- `handleConn(conn, err)`
  接到新連線後：
  - 建 socket
  - 做 allow IP 檢查
  - 加入 `Sockets`
  - 啟 session loop

- `createAndStartSocket(conn)`
  建立 socket 並 `Start()`。

- `allowConn(sock)`
  檢查遠端 IP 是否符合 allow list。

- `addSocket(sock, err)`
  把 socket 註冊進 `Sockets`。

- `startSocketSession(sock)`, `runSocketSession(sock)`
  每條連線的生命週期管理：
  - `Connect`
  - `sock.Wait()`
  - `Disconnect`

- `Kick(id)`
  依 socket ID 強制斷線。

- `SendAll`, `Send`, `SendByID`, `Get`, `GetByID`, `GetRandom`, `AddNewMsg`, `GetMsgFactory`, `GetMsgHandler`, `Async`, `SetSocketFactory`, `SetEventDispatcher`, `SetCoder`, `SetTransport`
  都是把 `serviceBase` 的能力轉出來。

## `client.go`

### 結構

- `type Client struct`
  `serviceBase` 之上的主動撥號端，增加：
  - `reConnectSecond`
    舊版固定秒數重連配置
  - `reconnectPolicy`
    新版策略介面
  - `allowedIPs`
    目前主要用於地址檢查輔助
  - `dialIPInfos`
    從設定讀出的目標位址清單

### 函式

- `newClient(...)`
  建立 client。

- `CheckAndGetAllowedIPs(ip)`
  從 `allowedIPs` 中找匹配地址，常見於檢查來路。

- `Connect(sock)`, `Disconnect(sock)`
  與 `Server` 類似，差別是 log 字段偏向 dial 端。

- `Init(s)`, `Destroy(s)`
  client 啟動/結束 lifecycle。

- `SetReConnectSecond(value)`
  設定固定重連間隔。

- `SetReconnectStrategy(policy)`
  指定完整重連策略。

- `reconnectStrategy()`
  回傳當前有效策略：
  - 先看 `reconnectPolicy`
  - 否則看 `reConnectSecond`
  - 最後預設 5 秒 `FixedBackoff`
  這裡回傳的是 `pkg/tcpclient` 內部的 strategy（原 `pkg/reconnect.Strategy`，已整併）。

- `Start()`
  入口。解析 dial addresses、建立 socket 容器、開始 client 主流程。

- `start()`
  啟送包 goroutine 與 `runClient()`。

- `Close()`, `close()`
  關閉 client 與所有 socket。
  現在也會停止後續重連。

- `prepareDial()`
  從設定讀出 `dial.<service>.*.ip`。

- `runClient()`
  client 主流程：
  - `Init`
  - 啟動單一 `dialLoop`
  - 等 goroutine 結束
  - `Destroy`

- `startDialLoop()`, `dialLoop()`
  單一重連迴圈，失敗時按順序切換下一個候選位址。

- `dialWithRetry(policy)`
  持續重試撥號直到 service 關閉。多個候選位址時，依序 failover。

- `dialOnce(addr, policy, sock, attempt)`
  單次撥號：
  - 失敗就 sleep/backoff
  - 成功就進 `handleDialSuccess`

- `handleDialSuccess(...)`
  成功後：
  - 舊 socket 若存在先關掉
  - 建新 socket
  - 加入 `Sockets`
  - 跑 session
  - session 結束後依策略決定下次延遲

- `shouldStop()`
  判斷 client 是否已關閉。

- `waitOrStop(delay)`
  等待重試延遲，但若 client 關閉則提早返回。

- `createAndStartSocket(conn)`, `addSocket(sock)`, `runSession(sock)`
  和 server 類似。

- `Kick(id)`
  依 ID 強制關閉對應 socket。

- 其餘 `Send*`、`Get*`、`AddNewMsg`、`Set*`
  都是轉發到 `serviceBase`。

## `tcpsocket/socket.go`

`Socket` 是最核心的一層，代表單條 TCP 連線。

### 結構與介面

- `type IPInfo struct`
  地址資訊：
  - `ID`
    這個目標節點的邏輯 ID
  - `TCPAddr`
    解析後的 TCP 位址

- `type ISocket interface`
  給 service / handler 操作 socket 的統一介面。

- `type Socket struct`
  主要欄位：
  - `key`
    每條連線的唯一字串 key
  - `id`
    業務邏輯上的節點/玩家 ID，可在握手後再設定
  - `conn`
    實際 net.Conn
  - `sendCh`
    socket 專屬送包 queue
  - `buffer`
    讀取用 buffer
  - `dispatcher`
    收到訊息後如何派發 handler
  - `coder`
    編解碼器
  - `IService`
    這條 socket 隸屬哪個 service
  - `msgFactory`
    根據 `MsgID` 建 message instance 的抽象

- `type messageFactory interface`
  封裝 `Create(enum.MsgID)`。

- `type serviceMessageFactory struct`
  預設實作，實際上就是去問 `IService.GetMsgFactory(id)`。

- `type FakerSocket struct`
  假 socket，主要用於測試/占位。

### 函式

- `NewSocket(...)`
  對外建立 socket 的工廠。

- `serviceMessageFactory.Create(id)`
  依 `MsgID` 找 factory 並建立 message instance。

- `GetConn`, `GetBuffer`, `GetKey`, `SetKey`, `GetID`, `SetID`, `SetData`, `GetData`, `RemoteAddr`, `LocalAddr`
  基本 getter/setter。

- `sessionLabel()`, `sessionValue()`
  內部 log helper，決定用 key 還是 ID 表示這條 session。

- `isClosedVal()`, `serviceName()`, `coderValue()`
  內部 helper。

- `logMessage(action, msg)`
  用 debug log 記錄 pack/unpack 訊息。

- `Start()`
  啟兩個 goroutine：
  - `receiveLoop()`
  - `sendLoop()`

- `Close()`, `close()`
  關閉 socket：
  - 標記 closed
  - 往 `sendCh` 塞 `nil` 讓 send loop 結束
  - 關 conn

- `Wait()`
  等 send/receive goroutine 收尾。

- `setReadDeadline()`, `setWriteDeadline()`
  套用讀寫 timeout。

- `receiveAsync()`, `sendAsync()`
  啟動 goroutine。

- `SendMsg(msg)`
  往 socket 送包 queue 塞 message。
  `nil` 代表結束送包 loop。

- `PushEvent(event)`
  把 handler 丟給 dispatcher。

- `Async(f)`
  在 socket waitgroup 下跑 background goroutine。

- `handleIncoming(id, b)`
  收到 payload 後：
  - 建 message instance
  - `Unpack`
  - 找 handler
  - 經 dispatcher 派發

- `handleOutgoing(msg)`
  送出前：
  - encode
  - write

- `newSocket(...)`
  真正建立 `Socket` struct，並初始化 `sendCh`、buffer、factory。

- `newSocketKey()`
  產生隨機 16-byte hex key。

- `receiveLoop()`
  持續呼叫 `receiveOnce()`，失敗後關 socket。

- `receiveOnce()`
  單次收包：
  - 設 read deadline
  - `coder.Decode`
  - 檢查 closed
  - 若 payload 有值就交給 `handleIncoming`

- `handleDecodeError(err)`
  EOF 以外的 decode error 會記 log。

- `sendLoop()`
  從 `sendCh.Out` 持續讀取訊息送出。

- `sendOnce(msg)`
  單次送包；若失敗就關 socket。

- `createMessage(id)`
  依 `MsgID` 建立對應 message instance。

- `getMessageHandler(id)`
  向 service 取出 handler。

- `unpackMessage(id, msg, b)`
  呼叫 `msg.Unpack`，失敗則記 log。

- `encodeOutgoing(msg)`
  `msg.Pack()` 後交給 coder 包成 wire format。

- `writeOutgoing(b)`
  真正寫進 `net.Conn`。

- `FakerSocket.*`
  全部只是 dummy 實作。

## `tcpsocket/sockets.go`

`Sockets` 是某個 service 底下的多連線容器。

### 介面與結構

- `type ISockets interface`
  定義 socket 集合該有的能力。

- `type Sockets struct`
  主要資料結構：
  - `sockets`
    key -> socket
  - `socketsByID`
    id -> socket
  - `activeKeys` / `activeIndex`
    為 `GetRandom()` 維護的活躍 key 索引

### 函式

- `newSockets()`
  建立容器。

- `AddNew(c)`
  新增 socket。
  若 key 空白會自動補 key。

- `Delete(key)`
  依 key 移除 socket。

- `DeleteByID(id)`
  依 ID 移除 socket。

- `Get(key)`
  依 key 查。

- `GetByID(id)`
  依 ID 查。
  若 `socketsByID` 過期，會回掃 `sockets` 重建索引。

- `findKeyByIDLocked(id)`
  lock 內部版的 ID -> key 查找。

- `indexSocketLocked(key, socket)`
  把 socket 加進：
  - 活躍 key 索引
  - ID 索引

- `removeSocketLocked(key)`
  從所有索引中刪除該 socket。

- `Set(key, c)`
  覆蓋或寫入 socket。

- `Clear()`
  清空索引，不主動關 socket。

- `SendAll(msg)`
  對所有 socket 廣播。
  這裡會把原訊息 `Pack()` 成 payload，再包成 `MsgPreprocess` 給每個 socket，避免重複做完整 struct pack。

- `Send(key, msg)`
  對單一 key 發送。

- `SendByID(id, msg)`
  對單一 ID 發送。

- `Close()`
  關掉全部 socket，並清掉所有索引。

- `GetRandom()`
  從活躍 key 中隨機挑一條 socket。

- `IsNil(key)`
  只是 `Get(key) == nil` 的包裝。

- `packWithPool(b)`
  payload copy helper，目前在這份程式碼裡沒有實際被主流程使用。

## `tcpsocket/coder.go`

### 介面與結構

- `type MessageCoder interface`
  最基本編解碼介面：
  - `Encode`
  - `Decode`

- `type PooledMessageCoder interface`
  支援 pooled encode。

- `type PooledMessageDecoder interface`
  支援 pooled decode。

- `type Coder struct`
  預設實作。

Wire 格式、範例與限制統一見[framecodec 封包格式](framecodec封包格式.md)，本文件只保留 `tcpsocket.Coder` 的 API。

### 函式

- `DefaultCoder()`
  取目前全域預設 coder。

- `SetDefaultCoder(c)`
  替換全域預設 coder。

- `(r *Coder) Encode(...)`
  普通編碼版本，內部實際呼叫 `EncodePooled()` 再 copy 一份。

- `(r *Coder) EncodePooled(...)`
  用 packet pool 建立封包，減少暫時物件。

- `(r *Coder) DecodePooled(reader)`
  pooled 版解碼，回傳 release func。

- `(r *Coder) Decode(reader, b)`
  一般解碼：
  - 讀 size
  - 讀 msg id
  - 讀 payload

## `msg_codec.go`

這些都是訊息 pack / unpack 的輔助函式。

- `packMsg(v)`
  把 struct pack 成 bytes。

- `unpackMsg(v, b)`
  從 bytes unpack 到 struct。

- `packMsgWith(...)`, `unpackMsgWith(...)`
  可指定 pack/unpack 策略的泛型 helper。

- `packMsgTo(v, buf)`
  寫進 buffer。

- `unpackMsgFrom(v, buf)`
  從 buffer 讀出。

## `pkg/msgpool`

- `MsgPool[T]`
  泛型訊息池。

- `MsgBase[T]`
  訊息基底型別，共用 `GetMsgID` / `Pack` / `Unpack` / `Put`。

- `NewMsgPool(newFn)`
  建立 message pool。

這是 handler 端最常碰到的 API。

## `config.go`

- `ServerOptionsFromConfig()`
  共用的啟動 helper，從 viper 讀取 TCP runtime 設定：
  - `tcp.rate_limit_per_second`
  - `tcp.rate_limit_burst`
  - `tcp.max_connections`
  - `tcp.socket_send_queue_depth`

  只接受正值；`0` 或負值會回落到預設值，避免壞設定把 rate limit / 連線上限弄壞。

- `resolveListenAddr(appID, serviceName)`
  從 viper 讀取：
  - `<appID>.listen.<serviceName>.ip`

- `resolveDialAddrs(appID, serviceName)`
  從 viper 讀取：
  - `<appID>.dial.<serviceName>.*.ip`

  回傳的是 `[]IPInfo`，其中 `ID` 來自設定 key。
  壞 entry 會被跳過；只有全部都壞掉時才回錯，避免單筆髒設定拖垮整個 client 啟動。

## `tcpclient` 的 reconnect

### 介面與結構

- `type Strategy interface`
  client 斷線後如何決定下次重試間隔。

- `type FixedBackoff struct`
  固定間隔。

- `type ExponentialBackoff struct`
  指數退避。

### 函式

- `FixedBackoff.NextDelay()`
  無論重試幾次都回同一個 interval。

- `FixedBackoff.Reset()`
  無狀態，什麼都不做。

- `ExponentialBackoff.NextDelay()`
  依指數退避規則計算，並受最大值限制。

- `ExponentialBackoff.Reset()`
  無狀態，什麼都不做。

- `NewDefaultReconnectStrategy()`
  建立預設的指數退避策略。

- `NewExponentialBackoff(base, max, multiplier, jitter)`
  建立可自訂參數的指數退避策略。

## `transport.go`

### 介面與結構

- `type Transport interface`
  抽象 `Listen` / `Dial`。

- `type netTransport struct{}`
  預設實作，直接呼叫 `net.Listen` / `net.Dial`。

### 函式

- `(netTransport) Listen(...)`
  真正 listen。

- `(netTransport) Dial(...)`
  真正 dial。

- `SetTransport(t)`
  替換全域 transport。
  常用於測試，把真網路換成 fake transport。

## 你最常用的入口

如果你是寫上層業務，大多只會直接碰這些：

- `tcpregistry.NewRegistry()`
- `registry.NewService(...)`
- 各 handler 套件內的 local `addMsg(...)`
- `svc.Send(...)`
- `svc.SendByID(...)`
- `svc.GetByID(...)`
- `svc.GetRandom()`
- `sock.SendMsg(...)`
- `sock.SetID(...)`

## 比較內部、通常不用直接碰的部分

- `serviceBase`
- `newServer`, `newClient`
- `Socket.receiveLoop/sendLoop`
- `Sockets.indexSocketLocked/removeSocketLocked`
- `packMsgWith`, `unpackMsgWith`
- `serviceMessageFactory`

這些比較像 runtime 內部實作細節。

## 目前值得注意的設計點

- `Socket` 的 handler 派發是靠 `dispatcher`，所以收到包後不一定立刻在同一 goroutine 執行。
- `Sockets.GetByID()` 會容忍 ID 索引過期，必要時會回掃全部 socket 重建。
- `Client` 現在 `Close()` 後會停止重連，這對測試與程序關閉很重要。
- `SendAll()` 會把 payload 轉成 `MsgPreprocess` 再送，避免重複對同一 struct 做完整 pack。
- `AddNewMsg()` 是用 `MsgID` 直接做陣列索引，所以 `MsgID` 必須落在 `uint16` 範圍內。

## 建議閱讀順序

如果你要真的讀懂這個套件，建議順序是：

1. `service.go`
2. `service_base.go`
3. `server.go` / `client.go`
4. `tcpsocket/socket.go`
5. `tcpsocket/sockets.go`
6. `tcpsocket/coder.go`
7. `config.go` / `reconnect.go` / `transport.go`

這樣最容易從外層入口一路看到封包收發細節。
