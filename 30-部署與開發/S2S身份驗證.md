# S2S 身份驗證

所有遊戲服務間的 TCP 連線均可使用「TLS＋節點 token」。呼叫端先驗證接收端憑證，再透過加密連線送出本機 ID 與 token；不需要客戶端憑證。

| 連線 | 接收端服務名稱 | 接收端授權清單 | 呼叫端身份 |
| --- | --- | --- | --- |
| Gate→Central（60002） | gate | gate-tokens.json | GateID＋gate-central.token |
| Gate→World（61001） | gate | gate-tokens.json | GateID＋gate-world.token |
| Gate→Chat（63001） | chat | gate-tokens.json | GateID＋gate-chat.token |
| World→Central（60001） | world | world-tokens.json | WorldID＋world-central.token |

每台 Gate 依目的服務持有三組獨立 token：Central、World、Chat。每台 World 持有自己的 Central token。每組 token 使用 32 隨機位元組產生；不同節點、不同目的服務不共用。重新連線沿用該組 token，不是每次握手重新產生。

接收端授權檔只保存 `sha256:` 加上 64 位十六進位 SHA-256 摘要。呼叫端持有原始 token，透過 TLS 傳送；接收端計算摘要並固定時間比較。摘要不能直接當作 bearer token 登入。

預設 `s2s_auth.enabled: false`，維持開發流程。啟用時 Central、Gate、World、Chat 都必須設定 `s2s_auth.enabled: true`，或 `S2S_AUTH_ENABLED=true`。已啟用程序須具備所有必要連線機密；Gate 僅在設定 Chat dial 時要求 Chat token。驗證失敗不退回明文，錯誤開關值、缺失機密或明文授權清單會回傳啟動錯誤。

## 連線規則

1. TLS 1.3 與 token 驗證共用五秒截止時間；通過才建立遊戲 socket。
2. 每個接收端程序、每種角色的一個節點 ID，只能有一條有效連線。重複登入拒絕新連線、保留舊連線；清理尚未完成也拒絕重連。
3. socket ID 由驗證結果設定；ConnectInfo 不得改成其他身份。登入、同步、恢復驗證及 World 轉送的來源身份也必須符合連線。
4. 斷線等待正在執行的訊息、移除 socket 並完成斷線清理後，才釋放節點 ID；舊連線的排隊訊息會丟棄。驗證客戶端同樣阻擋舊連線訊息。
5. 每份清單禁止不同節點共用 token；日誌不顯示 token。
6. 接收端每秒檢查授權檔內容摘要，兼容 Docker／跨系統目錄掛載；不合法更新保留舊設定。撤銷或 token 改變時，關閉受影響節點的連線，其他節點不受影響。`{}` 撤銷該清單全部節點，`null` 不合法。

同一 Gate 可同時連到 Central、World、Chat。單連線限制作用於各接收端程序；多程序叢集的全域排他需共享租約。網路斷線尚未偵測時，新連線須等待既有 heartbeat/read timeout 清理。token 外洩時，先連入者仍可能冒充節點。

接收端清單外洩不會直接取得可登入的原始 token；每目的服務隔離也讓 Chat 的 token 無法登入 Central 或 World。Gate 主機被入侵時，其三組原始 token 仍可能外洩；接收端主機被入侵時，也可能攔截後續送到它的 token。此方案仍需要 TLS、主機權限與安全亂數。

## 設定

各服務範本與 `config/dev`、`config/docker`、`docker/prod/config` 已提供以下設定。原本 Gate→Central 的平面設定保留；新增連線使用子區段。若已啟用舊版驗證，升級前須補齊新增區段及機密。

Central：

```yaml
s2s_auth:
  enabled: true
  allowed_cidrs: []
  cert_file: /run/secrets/central.crt
  key_file: /run/secrets/central.key
  tokens_file: /run/secrets/gate-tokens.json
  world:
    allowed_cidrs: []
    cert_file: /run/secrets/central.crt
    key_file: /run/secrets/central.key
    tokens_file: /run/secrets/world-tokens.json
```

Gate：

```yaml
id: 2001
s2s_auth:
  enabled: true
  ca_file: /run/secrets/ca.crt
  token_file: /run/secrets/gate-central.token
  server_name: central.internal
  world:
    ca_file: /run/secrets/ca.crt
    token_file: /run/secrets/gate-world.token
    server_name: world.internal
  chat:
    ca_file: /run/secrets/ca.crt
    token_file: /run/secrets/gate-chat.token
    server_name: chat.internal
```

World：

```yaml
id: 1001
s2s_auth:
  enabled: true
  allowed_cidrs: []
  cert_file: /run/secrets/world.crt
  key_file: /run/secrets/world.key
  tokens_file: /run/secrets/gate-tokens.json
  central:
    ca_file: /run/secrets/ca.crt
    token_file: /run/secrets/world-central.token
    server_name: central.internal
```

Chat：

```yaml
s2s_auth:
  enabled: true
  allowed_cidrs: []
  cert_file: /run/secrets/chat.crt
  key_file: /run/secrets/chat.key
  tokens_file: /run/secrets/gate-tokens.json
```

清單是 JSON 物件，例如 `{"2001":"sha256:<64位十六進位摘要>"}`；其中角括號部分須換成真實摘要。不得填入原始 token。原始 token 長度限制為 43 至 256 位元組，須用安全亂數產生器。憑證 DNS SAN 必須包含客戶端的 `server_name`；TCP 使用內部 IP 也能以名稱驗證，不能關閉驗證。各鍵可用大寫環境變數覆寫，例如 `S2S_AUTH_WORLD_SERVER_NAME`。

各監聽區段可設定 `allowed_cidrs`，例如 `["10.1.2.3/32", "fd00::1/128"]`。空清單不限制來源。依真實 TCP 來源 IP 在 TLS 前檢查，不信任轉發標頭；修改需重啟對應接收端。Docker、NAT 應填接收端實際看到的來源。IP 限制不能取代 token 驗證。

## Docker 開發

安裝 OpenSSL 後，從專案根目錄執行。腳本拒絕覆寫既有目錄，不顯示 token。

```powershell
./scripts/check-s2s-dev-secrets.ps1
./scripts/new-s2s-dev-secrets.ps1 -GateIDs 2001,2002 -WorldIDs 1001,1002
$env:S2S_SECRET_DIR = (Resolve-Path secrets/s2s).Path.Replace('\', '/')
$env:S2S_GATE_ID = '2001'
$env:S2S_WORLD_ID = '1001'
docker compose -f docker/compose/compose.full.yml -f docker/compose/compose.s2s-auth.yml config --quiet
docker compose -f docker/compose/compose.full.yml -f docker/compose/compose.s2s-auth.yml up -d --build
```

掛載的 Gate、World ID 必須符合該服務設定中的 `id`；環境變數 `S2S_GATE_ID`、`S2S_WORLD_ID` 只選擇機密目錄，不會改應用程式 ID。多台節點需各自的容器與 ID／網路設定。

```text
secrets/s2s/
├── ca.crt、ca.key                    # 管理端，CA 私鑰不掛到服務
├── world-gate-tokens.json            # 管理端，Gate→World 摘要來源
├── central/
│   ├── central.crt、central.key
│   ├── gate-tokens.json              # Gate→Central 摘要
│   └── world-tokens.json             # World→Central 摘要
├── gate-2001/
│   ├── ca.crt
│   └── gate-central.token、gate-world.token、gate-chat.token
├── world-1001/
│   ├── world.crt、world.key、ca.crt
│   ├── world-central.token
│   └── gate-tokens.json
└── chat/
    ├── chat.crt、chat.key
    └── gate-tokens.json
```

各服務只掛載自己的目錄。目錄掛載能看到原子替換後的新檔案，單檔掛載可能仍指向舊檔案。產生器僅供開發；各 World 開發副本共用測試服務憑證，但節點 token 獨立。正式環境應限制宿主機權限、獨立配置私鑰並由受控系統簽發。舊版明文／共用 token 格式不再接受。開發環境請先停止啟用驗證的服務，在新目錄重新產生機密，切換掛載並一起重建容器；不要直接把舊共用 token 複製成三份。正式環境需事先配置獨立 token、其摘要、信任 CA 與憑證，安排協調切換。

## 新增、輪替、撤銷

```powershell
./scripts/update-s2s-gate-secret.ps1 -Action Add -GateID 2003
./scripts/update-s2s-gate-secret.ps1 -Action Rotate -GateID 2001
./scripts/update-s2s-gate-secret.ps1 -Action Rotate -GateID 2001 -Destination World
./scripts/update-s2s-gate-secret.ps1 -Action Revoke -GateID 2002 -Destination Chat
./scripts/update-s2s-gate-secret.ps1 -Action Revoke -GateID 2003
./scripts/update-s2s-gate-secret.ps1 -Action Add -Role World -NodeID 1003
./scripts/update-s2s-gate-secret.ps1 -Action Rotate -Role World -NodeID 1001
./scripts/update-s2s-gate-secret.ps1 -Action Revoke -Role World -NodeID 1003
```

使用 `-SecretDirectory <目錄>` 可指定機密根目錄。`-Destination Central|World|Chat` 只操作該目的服務；預設 `All` 操作 Gate 三個目的服務。World 僅接受 Central／All。撤銷可重複執行，不會重新授權已撤銷的連線。

新增時可省略 ID，使用 `-PassThru` 取得配置結果：

```powershell
$node = ./scripts/update-s2s-gate-secret.ps1 -Action Add -Role Gate -PassThru
$node.NodeID
```

Gate 從 2001、World 從 1001 起分配，管理端 `node-ids.json` 保存水位；退役、發布失敗均不回收 ID。明確指定 ID 仍可使用，新增更高 ID 會提高水位。配置服務的 `id` 後才啟動；這不是自動開 VM 或建立容器。管理目錄與水位檔須一起備份，所有分配都透過同一管理端。

腳本以檔案鎖序列化操作；分別更新 Central、Chat 的 Gate 摘要清單，Gate→World 的摘要更新管理端 `world-gate-tokens.json` 並同步所有 `world-*` 目錄。新 World 複製目前的 World 專用 Gate 摘要。World 自身的 token 摘要只更新 Central 的 World 清單。

- 新增節點後，設定 ID、掛載及網路位址，再啟動節點；接收端不需重啟。
- 輪替先寫本機 token，再逐檔發布接收端摘要清單；發布失敗會還原已發布清單與本機 token。這不是跨機器交易，執行途中可能短暫拒絕連線，還原時也可能中斷受影響節點。
- Gate 的預設 All 輪替／撤銷影響三種接收端；指定 Destination 只影響該目的服務。World 輪替／撤銷只影響 World→Central；停用整台 World 還須停止程序並移除 Gate 的目的位址。
- 撤銷保留本機 token 檔案，接收端會拒絕該 token。各 TCP 客戶端重連時讀取新 token 與 CA；Chat 也會自動重連。操作可能影響玩家登入態，應安排適當時機。
- 各接收端在新握手讀取憑證與私鑰，換發不需重啟。兩檔短暫不一致時，新連線可能被拒絕。
- CA 輪替先分發新、舊 CA 信任檔，再換發各服務憑證，最後移除舊 CA。

本腳本更新管理端來源；跨主機部署再執行下列維護入口，把各角色機密分發到實際接收端。

## 跨主機分發與每日維護

管理主機需要 PowerShell、Python 3.9 以上、OpenSSL、SSH。Linux 管理端使用 PowerShell 7（`pwsh`）；接收主機需要 SSH 與 Python 3.9 以上。管理端是唯一機密來源，來源目錄含 CA 私鑰與全部 token，須限制只有管理帳號可讀寫。服務主機只能收到自己的角色目錄，不能掛載管理根目錄。

複製 `docker/s2s-targets.example.json`，填入全部接收主機、節點 ID、絕對目錄及實際容器 UID／GID。每台 World 都要列出，才能更新它的 Gate 授權清單。先透過可信管道核對並加入 SSH 主機公鑰；分發使用 `StrictHostKeyChecking=yes`，不自動信任未知主機。可設定 `identity_file`、`known_hosts_file`、`port`；`sudo: true` 需要遠端允許指定管理帳號非互動執行發布。範例 UID 100／GID 101 須依映像確認。

```powershell
./scripts/invoke-s2s-maintenance.ps1 -SecretDirectory C:/private/pja-s2s -TargetsFile C:/private/s2s-targets.json
```

維護入口序列化排程，發布期間持有來源檔案鎖；不要直接執行底層 Python 分發／續期程式。分發只包裝固定角色檔案，檢查 SHA-256 摘要、大小及檔名；不傳 CA 私鑰。SSH 封包經標準輸入送入遠端，機密不出現在命令參數或日誌。遠端目錄權限 0700、檔案 0600，採原子替換並在失敗時還原。Windows 本機測試的 POSIX 權限不能取代 NTFS ACL。

- 預設輪替超過 30 天的有效 token，可用 `-TokenMaxAgeDays` 調整；已撤銷目的授權不重新開通。
- 預設在到期前 30 天換發 90 天服務憑證，保留 DNS SAN，產生新私鑰並驗證 CA。各 World 續期後使用獨立私鑰。非 DNS SAN、CA 即將到期或簽發失敗會停止，不降低名稱驗證或自動更換 CA。
- 每次維護均分發目前版本。部分主機失敗後再次執行，不會把剛更新的 token 再輪替；跨主機發布不是交易，期間可能短暫重連失敗。新增／撤銷後也要執行維護入口，才能同步遠端。
- 憑證與私鑰逐檔替換，新握手可能短暫失敗後重試；已建立連線不因服務憑證續期而關閉。CA 輪替仍需人工協調新舊信任。
- 腳本失敗回傳非零碼；實際告警目的地待正式環境選定後接入。

Windows 原生排程預設只輸出 XML，供檢閱；加 `-Install` 才註冊每日 03:00 工作：

```powershell
./scripts/new-s2s-maintenance-task.ps1 -SecretDirectory C:/private/pja-s2s -TargetsFile C:/private/s2s-targets.json
# 確認正式管理帳號、路徑及權限後，再加 -Install。
```

排程以目前帳號 S4U 背景執行，OpenSSL／SSH 必須在該帳號的 PATH；SSH 使用本機私鑰，不能依賴互動密碼、登入時的 agent 或加密檔案。註冊可能需要管理員權限。Linux 可使用管理帳號的原生 cron，例如每天 03:00 執行：

```cron
0 3 * * * /usr/bin/pwsh -NoProfile -NonInteractive -File /srv/pja/scripts/invoke-s2s-maintenance.ps1 -SecretDirectory /srv/private/pja-s2s -TargetsFile /srv/private/s2s-targets.json -Python /usr/bin/python3 >> /srv/private/s2s-maintenance.log 2>&1
```

排程部署在能存取 CA 的管理主機，不能放在 Gate 容器內。正式主機尚未選定，本次只提供並檢查配置，不安裝正式排程。

## 可重跑驗收

```powershell
./scripts/check-s2s-dev-secrets.ps1
python -B scripts/check-s2s-distribution.py --source C:/private/test-s2s
python -B scripts/check-s2s-maintenance.py
./scripts/check-s2s-docker.ps1
```

分發檢查需要以開發產生器建立含 Gate 2001、World 1001 的隔離來源。維護檢查自行建立與刪除暫存機密，驗證有效授權輪替、撤銷不復活、續期簽章及分發失敗重試。完整 Docker 檢查建立專用 Compose 專案，啟動 12 個服務、不發布主機端口；確認四條 S2S、跨目的 token 拒絕、重複身份拒絕、單目的輪替重連及撤銷。預設結束關閉專用容器，保留測試資料卷與暫存機密供查核；此腳本會修改指定的來源，不能指向正式目錄。

2026-09-26 已通過完整 Docker 驗收、換發憑證後的實際 TLS 握手、SSH 容器分發與本機維護檢查。Windows 排程 XML 已成功輸出，尚未安裝／觸發正式主機排程。

## 部署範圍

本次已納入上述四條 TCP S2S；玩家→Gate 維持原有玩家驗證，HTTP、Redis、MySQL、RabbitMQ 使用各自既有的驗證機制。

本機／Docker 不需要 Google Cloud。通用 ID 分配、SSH 分發、token 輪替、憑證續期與排程配置已提供。依目前決定，GCP 資源、Secret Manager／IAM／VPC 防火牆及實際通知目的地暫不建立；worktree 分支不合併。此方案的排他仍以接收端程序為範圍，多程序共享租約未納入。
