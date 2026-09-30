# GoLand 本機開發

這份文件說明如何把依賴交給 Docker，並讓 `central` / `world` / `gate` / `record` / `user` 直接在本機用 GoLand 啟動與除錯。

## 目標

- 開發時：
  - `mysql` / `redis` / `rabbitmq` / `prometheus` / `grafana` 用 Docker 跑
  - `central` / `world` / `gate` / `record` / `user` 用 GoLand 本機跑
- 驗證整套容器時：
- 使用 [[Docker全堆疊]]
- 上線時：
  - 以 Docker image 與 Docker compose 方式部署

## 啟動開發依賴

先在專案目錄執行：

```bash
./scripts/docker-deps-up.sh
```

或直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode dockerdepsup
```

或直接：

```bash
docker compose -f docker/compose/compose.deps.yml up -d
```

這會啟動：

- `mysql`
- `redis`
- `rabbitmq`
- `prometheus`
- `grafana`

依賴對外入口：

- MySQL: `127.0.0.1:13306`
- Redis: `127.0.0.1:16379`
- RabbitMQ: `127.0.0.1:25672`
- Prometheus: [http://localhost:39090](http://localhost:39090)
- Grafana: [http://localhost:33000](http://localhost:33000)

這組埠刻意避開 full stack 的 `3000 / 9090`，因此可與 [[Docker全堆疊]] 並存。

Grafana 預設帳密：

- `admin`
- `admin`

## 本機設定檔

GoLand 本機執行請使用：

- [central.yaml](../../config/dev/central.yaml)
- [world.yaml](../../config/dev/world.yaml)
- [gate.yaml](../../config/dev/gate.yaml)
- [record.yaml](../../config/dev/record.yaml)
- [user.yaml](../../config/dev/user.yaml)

這些設定檔與 Docker 版最大的差異是：

- MySQL / Redis 改走宿主機埠：
  - `127.0.0.1:13306`
  - `127.0.0.1:16379`
- RabbitMQ 改走宿主機埠：
  - `127.0.0.1:25672`
- 服務間通訊改走本機 TCP：
  - `127.0.0.1:60001`
  - `127.0.0.1:60002`
  - `127.0.0.1:61001`
  - `127.0.0.1:62001`
- `world` 會把一般事件異步送到 `record`

## GoLand Run/Debug Configuration

GoLand 開啟的專案根目錄應是 `PJA-Project`，`PJA-Server` 是其中的服務目錄。每個服務建立一個 `Go Application` configuration，設定如下：

- `Central`
- Package：`local/server/cmd/central`
- `World`
- Package：`local/server/cmd/world`
- `Gate`
- Package：`local/server/cmd/gate`
- `Record`
- Package：`local/server/cmd/record`
- `User`
- Package：`local/server/cmd/user`

所有 configuration 共用以下設定：

- Working directory：`$PROJECT_DIR$/PJA-Server`
- Module：`Work`
- Run kind：`Package`
- Program arguments：`-config config/dev/<service>.yaml`

例如 Central：

```text
Package: local/server/cmd/central
Working directory: $PROJECT_DIR$/PJA-Server
Program arguments: -config config/dev/central.yaml
```

設定必須勾選 `Store as project file`，設定檔應保存於 GoLand 實際專案根目錄的：

```text
PJA-Project/.idea/runConfigurations/
```

不要放在 `PJA-Server/.idea/runConfigurations/`；若 GoLand 開啟的是上層 `PJA-Project`，它不會讀取服務子目錄下的 `.idea`。

不要使用 GoLand 自動產生的 `go build example.com/project/...` 設定。這類設定通常把 Working directory 設成 `PJA-Server/cmd/<service>`，且沒有 `-config`，會造成 `bootstrap: no config file found`。

每個 configuration 都加這個環境變數：

```text
DEBUG_ALLOW_CIDRS=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```

再分別指定 `APP_CONFIG`：

- `Central`
  - `APP_CONFIG=E:/project/config/dev/central.yaml`
- `World`
  - `APP_CONFIG=E:/project/config/dev/world.yaml`
- `Gate`
  - `APP_CONFIG=E:/project/config/dev/gate.yaml`
- `Record`
  - `APP_CONFIG=E:/project/config/dev/record.yaml`
- `User`
  - `APP_CONFIG=E:/project/config/dev/user.yaml`

`DEBUG_ALLOW_CIDRS` 的目的，是讓 Docker 裡的 Prometheus 可以抓本機服務的 `/metrics`。

## 建議啟動順序

1. 啟動 Docker 依賴
2. 在 GoLand 啟動 `Central`
3. 在 GoLand 啟動 `Record`
4. 在 GoLand 啟動 `World`
5. 在 GoLand 啟動 `Gate`
6. 在 GoLand 啟動 `User`

順序與一般本機啟動相同，因為：

- `World` 會主動連 `Central`
- `Record` 會主動連 RabbitMQ 與 MySQL
- `Gate` 會主動連 `Central` 與 `World`
- `User` 會主動連 `Gate`

## 驗證流程

1. 打開 Grafana，看 `Game Backend Overview`
2. 在 `User` 的執行視窗輸入：

```text
login 0 alice
```

3. 確認：
- `User` 出現 `login success`
- `Gate / Central / World / Record` 各自有登入處理 log
- Grafana / Prometheus 有 metrics

## 何時改用 full stack Docker

以下情境改用 [[Docker全堆疊]]：

- 驗證容器網路行為
- 驗證完整容器化部署
- 驗證 image build 或 compose 啟動問題
- 模擬接近上線的執行方式

## 上線建議

開發與上線不要共用同一份設定檔：

- 開發用：
  - `config/dev/*.yaml`
  - `docker/compose/compose.deps.yml`
- Docker 全堆疊驗證用：
  - `config/dev/*.yaml`
  - `docker/compose/compose.full.yml`

正式上線時，建議再額外準備 production 專用的：

- image tag 策略
- secrets 管理
- volume / 備份策略
- 對外 port 與防火牆規則
- production compose 或更正式的部署編排
