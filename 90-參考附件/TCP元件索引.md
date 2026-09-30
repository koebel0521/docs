# TCP 元件索引

本頁提供 TCP 基礎設施的閱讀順序；完整結構、介面與函式細節見[TCP 封包詳細參考](TCP封包參考.md)。

| 元件 | 責任 | 代碼入口 |
| --- | --- | --- |
| `tcpserver` | 監聽連線、服務生命週期與收發協調 | `pkg/tcpserver` |
| `tcpclient` | 連線、重連與請求生命週期 | `pkg/tcpclient` |
| `tcpsocket` | socket、frame coder 與封包讀寫 | `pkg/tcpsocket`、`pkg/framecodec` |
| `tcpruntime` | 對外最小 runtime 介面 | `pkg/tcpruntime` |
| `tcpregistry` | 服務註冊與查找 | `pkg/tcpregistry`、`pkg/servicefinder` |
| `tcpdiag` | 協定記錄與診斷資料 | `pkg/tcpdiag`、`pkg/debug` |

## 建議順序

1. 先看[系統導讀](../00-導讀/系統導讀.md)了解服務拓撲。
2. 再看[連線與登入流程](../10-核心流程/連線與登入流程.md)了解實際呼叫鏈。
3. 需要封包格式時看[framecodec 封包格式](framecodec封包格式.md)。
4. 需要 API 細節時查[TCP 封包詳細參考](TCP封包參考.md)。
