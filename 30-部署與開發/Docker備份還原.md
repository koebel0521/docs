# Docker 備份還原

這份文件描述目前 production Docker 骨架下的 MySQL / Redis 備份與還原方式。

## 範圍

目前提供 4 個腳本：

- [docker-backup-mysql.sh](../../scripts/docker-backup-mysql.sh)
- [docker-restore-mysql.sh](../../scripts/docker-restore-mysql.sh)
- [docker-backup-redis.sh](../../scripts/docker-backup-redis.sh)
- [docker-restore-redis.sh](../../scripts/docker-restore-redis.sh)

這些腳本預設對應：

- `docker/compose/compose.prod.yml`
- `.env.prod`（以 `scripts/lib/envfile.sh` 的 `env_get` 依 compose dotenv 規則逐鍵讀，不 `source`：檔案不需是合法的 bash，`MYSQL_DSN=...@tcp(127.0.0.1:3306)/...` 可不加引號，引號、行內註解、`export` 前綴與 CRLF 都與 compose 的解析一致）
- `project-prod_prod-redis-data` volume

## MySQL 備份

```bash
./scripts/docker-backup-mysql.sh
```

預設輸出到：

- `~/pja-backups/mysql/mysql-YYYYMMDD-HHMMSS.sql.gz`（repo 外；權限 0600、目錄 0700，腳本以 `umask 077` 建立）

`BACKUP_SKIP_RECORD_EVENTS=1` 可排除全部 `record_events_*` 日分表（預設 0＝一併備份；事件流水另行保存或可接受遺失時使用，停機備份會因此縮短）。只保留最近 `BACKUP_KEEP` 份（預設 20，`0` 表示不清理）；備份與正式資料同機，不輪替會慢慢寫滿磁碟，需要更長保存或異地備份請自行複製。CD 部署（[Docker發版流程](Docker發版流程.md)）會在服務停止後多備份一份，檔名帶 `-final`（`TIMESTAMP` 後綴），也算在保留份數內。

可覆寫：

```bash
ENV_FILE=.env.prod BACKUP_DIR=/srv/backups/mysql ./scripts/docker-backup-mysql.sh
```

## MySQL 還原

```bash
CONFIRM_RESTORE=yes ./scripts/docker-restore-mysql.sh ~/pja-backups/mysql/mysql-20260324-120000.sql.gz
```

或直接走維護者本機的統一入口（`scripts/private/` 不進版控，其他人以上面的公開腳本為準）：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerrestoremysql -DockerRestoreMysqlInputFile ~/pja-backups/mysql/mysql-20260324-120000.sql.gz
```

注意：

- 這會直接把 dump 匯回 `MYSQL_DATABASE`；沒有 `CONFIRM_RESTORE=yes` 腳本拒絕執行，還原前會先備份現況
- 還原前應先確認目標環境是否允許覆寫資料
- 最好先在 staging 演練一次

## Redis 備份

```bash
./scripts/docker-backup-redis.sh
```

預設輸出到：

- `~/pja-backups/redis/redis-YYYYMMDD-HHMMSS.tar.gz`（repo 外；權限 0600）

這份 tar 會包含 Redis volume 內的資料檔。同樣只保留最近 `BACKUP_KEEP` 份（預設 20，`0` 表示不清理）。

## Redis 還原

```bash
CONFIRM_RESTORE=yes ./scripts/docker-restore-redis.sh ~/pja-backups/redis/redis-20260324-120000.tar.gz
```

或直接走維護者本機的統一入口（`scripts/private/` 不進版控，其他人以上面的公開腳本為準）：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerrestoreredis -DockerRestoreRedisInputFile ~/pja-backups/redis/redis-20260324-120000.tar.gz
```

注意：

- 腳本會先 `stop redis`
- 清空 Redis data volume 後再展開備份
- 還原完成後再把 Redis 起回來

## 重要限制

1. Redis restore 是覆蓋式還原
2. MySQL restore 也是直接匯入目標 DB
3. 這些腳本適合單機 production 骨架
4. 若你之後改成受管 MySQL / Redis，腳本也要改

## 建議操作順序

1. 部署前先做一次 backup
2. 先執行 migration job，再更新服務 image
3. 升級後跑 smoke test
4. 若 smoke test 失敗，先 rollback image tag，不要 rollback database schema
5. 若資料也已受損，先隔離流量，再從已驗證的 backup/snapshot restore

database schema 的向前遷移是 release manifest 的一部分。舊版 server rollback 應保持新 schema 相容；需要測試歷史版本時，請使用 production snapshot 的隔離副本。

若只想先把兩份備份打出來，也可以直接走維護者本機的統一入口（`scripts/private/` 不進版控，其他人以上面的公開腳本為準）：

```powershell
powershell -ExecutionPolicy Bypass -File ./scripts/private/p0-daily-check.ps1 -Mode dockerbackup
```

## 建議下一步

若要把備份這條線再補完整，我建議下一批做：

1. backup 目錄輪替策略
2. 自動化排程
3. restore 演練文件
4. 備份完整性檢查
