# Docker 全堆疊

這份文件描述如何用單一 `docker compose` 啟動完整遊戲專案。
若你要和 production 同機並跑，請改用 [compose.debug.yml](../../docker/compose/compose.debug.yml)。

- `mysql`
- `redis`
- `central`
- `world`
- `record`
- `gate`
- `user`
- `rabbitmq`
- `prometheus`
- `grafana`

## 使用流程

## World 資料檔

World 會從容器內的 `config/data` 載入棋盤與效果資料；`compose.full.yml` 已將專案的 `config/data` 唯讀掛載到 `/app/config/data`。若缺少此掛載，World 會因找不到 `Shared/Effect.bytes` 而無法通過健康檢查。

1. 啟動整套服務
2. 確認容器與監控正常
3. attach `user` 容器做登入測試
4. 用 `docker logs` 或 Grafana 觀察整條鏈路
5. `record` 會吃 `world` 異步送出的事件流水
6. 改完程式後只重建有變更的服務
7. 結束時關閉堆疊

## 啟動

```bash
./scripts/docker-full-up.sh
```

或直接走統一入口：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/p0-daily-check.ps1 -Mode dockerfullup
```

或直接：

```bash
docker compose -f docker/compose/compose.full.yml up -d --build
```

debug 版本另有獨立檔案：

```bash
docker compose -f docker/compose/compose.debug.yml up -d --build
```

啟動後先確認容器狀態：

```bash
docker compose -f docker/compose/compose.full.yml ps
```

正常情況下應至少看到：

- `mysql` / `redis` 為 `healthy`
- `central` / `world` / `record` / `gate` / `user` / `rabbitmq` 為 `healthy`
- `prometheus` / `grafana` 為 `Up`

## 對外入口

- Gate user TCP: `localhost:62001`
- Central world TCP: `localhost:60001`
- Central gate TCP: `localhost:60002`
- World gate TCP: `localhost:61001`
- Record HTTP: `localhost:6070`
- RabbitMQ AMQP: `localhost:5672`
- RabbitMQ UI: [http://localhost:15672](http://localhost:15672)
- Grafana: [http://localhost:3000](http://localhost:3000)
- Prometheus: [http://localhost:9090](http://localhost:9090)

若是 debug stack，對外入口會改成：

- Gate user TCP: `localhost:16201`
- Central world TCP: `localhost:16001`
- Central gate TCP: `localhost:16002`
- World gate TCP: `localhost:16101`
- Record HTTP: `localhost:6070`
- RabbitMQ AMQP: `localhost:5672`
- RabbitMQ UI: [http://localhost:15672](http://localhost:15672)
- Lobby HTTP: `localhost:18080`
- Grafana: [http://localhost:13000](http://localhost:13000)
- Prometheus: [http://localhost:19090](http://localhost:19090)

## 監控

Prometheus 會直接抓容器網路內的：

- `central:6060`
- `gate:6061`
- `world:6062`
- `record:6065`
- `user:6063`

Grafana 會自動 provision `Game Backend Overview` dashboard。

## 驗證監控

啟動後可直接打開：

- Grafana: [http://localhost:3000](http://localhost:3000)
- Prometheus: [http://localhost:9090](http://localhost:9090)

Grafana 預設帳密：

- `admin`
- `admin`

可先確認：

- Grafana 已出現 `Game Backend Overview`
- Prometheus `up` 查詢可看到 `central` / `gate` / `world` / `record` / `user`

## User 容器使用方式

`user` 是互動式 client。啟動後可直接 attach：

```bash
docker attach project-user
```

目前最小可用指令是：

```text
login 0 alice
```

成功時可在 `user` 端看到類似 `login success` 的輸出。
其中 `0` 是數字形式的帳號類型；User 會依 `config/docker/user.yaml` 的 `tcp.<appID>.dial` 設定連線到 Gate。
World 啟動時會將 Redis 能力公告放在背景工作執行，不會阻塞應用程式事件迴圈；因此登入請求可正常由 World 回傳至 Gate。

若要離開 attach 又不停止容器，使用 Docker detach sequence：

```text
Ctrl-p Ctrl-q
```

## 觀察登入鏈路

測完登入後，可分別看各服務 log：

```bash
docker logs project-user --tail 100
docker logs project-gate --tail 100
docker logs project-central --tail 100
docker logs project-world --tail 100
docker logs project-record --tail 100
```

你應該能看到：

- `user` 收到登入結果
- `gate` 收到 user login 並轉送 `central`
- `central` 處理帳號登入與 world 分派
- `world` 接收登入並建立玩家狀態
- `record` 接收 world 事件並寫入流水

## 常用重建

若只改到部分服務，不需要整套重建。

例如只改 `gate` / `user` / `record`：

```bash
docker compose -f docker/compose/compose.full.yml up -d --build gate user record
```

整套重建：

```bash
docker compose -f docker/compose/compose.full.yml up -d --build
```

## 配置檔

容器內配置位於：

- [central.yaml](../../config/dev/central.yaml)
- [world.yaml](../../config/dev/world.yaml)
- [record.yaml](../../config/dev/record.yaml)
- [gate.yaml](../../config/dev/gate.yaml)
- [user.yaml](../../config/dev/user.yaml)

debug stack 仍使用同一組容器內 config，但會把 `DEBUG_PROTOCOL_RECORD=true` 打進容器環境，讓協定記錄預設開啟。

## 健康檢查

compose 會用各服務的 `/ready` 做 healthcheck，並在依賴 ready 後才啟動下游服務。

## 停止

```bash
docker compose -f docker/compose/compose.full.yml down
```

若連資料卷也要清掉：

```bash
docker compose -f docker/compose/compose.full.yml down -v
```

`down -v` 會刪掉 MySQL / Redis 資料卷，只適合要重置本機開發資料時使用。
