# `scripts/docker-restore-redis.sh`

## 作用
把 Redis 備份還原回 production Redis volume。

## 目的
在清掉壞資料後，快速把 volume 還原回來。

## 流程
1. 沒有設 `CONFIRM_RESTORE=yes` 就拒絕執行。
2. 確認 tar.gz 能完整讀出且不是空包。
3. 先把目前的 Redis 備份到 `PRE_RESTORE_BACKUP_DIR`（預設 `~/pja-backups/redis-pre-restore`），可用 `SKIP_PRE_RESTORE_BACKUP=1` 略過；Redis 未啟動時這步備份失敗，整個還原中止。
4. 停掉 Redis。
5. 用 `busybox` 把 tar.gz 解到 volume 內的暫存目錄，舊資料移到 `.restore-old` 後才換入新資料。
6. 再把 Redis 起回來。

解壓失敗時（尚未建立 `.restore-old`）會清掉暫存目錄、舊資料原封不動留在 volume 根目錄，Redis 維持停止；換入途中失敗才需要從 `.restore-old` 取回舊資料，不再留下空的 `.restore-old`。

volume 內已有上次失敗留下的 `.restore-old` 時腳本會拒絕繼續，那可能是唯一的舊資料，請先確認再手動處理。

## 主要參數
- 第一個位置參數：`redis-backup.tar.gz`
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`VOLUME_NAME`、`CONFIRM_RESTORE`（必須為 `yes`）、`PRE_RESTORE_BACKUP_DIR`、`SKIP_PRE_RESTORE_BACKUP`

## 相關文件
- [Docker備份Redis](./Docker備份Redis.md)
