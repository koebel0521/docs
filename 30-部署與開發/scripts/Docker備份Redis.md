# `scripts/docker-backup-redis.sh`

## 作用
匯出 production Redis volume。

## 目的
替 Redis 資料保留可回復的備份檔。

## 前置條件
- 需要 env 檔（`ENV_FILE`，預設 `.env.prod`），不存在直接報錯。
- Redis 容器必須在跑：腳本以 `docker compose exec` 對 `redis` 服務下指令（`COMPOSE_FILE` 預設 `docker/compose/compose.prod.yml`）。

## 流程
1. 決定備份目錄（預設 `~/pja-backups/redis`，repo 外）與輸出檔名。
2. 觸發 `BGSAVE` 並輪詢 `LASTSAVE` 等到完成（最多 `BGSAVE_TIMEOUT_SECONDS` 秒），確保 `dump.rdb` 是一致的快照；逾時直接失敗。
3. 用臨時 `busybox` 容器（映像釘 digest，版本一致性測試比對）把 volume 打成 `.tar.gz`；先寫 `.partial`，成功才改名。
4. 只保留最近 `BACKUP_KEEP` 份 `redis-*.tar.gz`（預設 20，`0` 表示不清理）。

## 主要參數
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`BACKUP_DIR`、`TIMESTAMP`、`OUTPUT_FILE`、`VOLUME_NAME`、`BGSAVE_TIMEOUT_SECONDS`（預設 120）、`BUSYBOX_IMAGE`、`BACKUP_KEEP`（須為非負整數，否則在備份前以 exit 2 失敗）

## 相關文件
- [Docker還原Redis](./Docker還原Redis.md)
