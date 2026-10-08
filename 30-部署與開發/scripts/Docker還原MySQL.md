# `scripts/docker-restore-mysql.sh`

## 作用
把 MySQL 備份還原回 compose 裡的 `mysql`。

## 目的
事故後快速把資料恢復回去。

## 流程
1. 讀 `.env.prod`。
2. 檢查輸入備份檔存在；沒有設 `CONFIRM_RESTORE=yes` 就拒絕執行。
3. `.gz` 備份先用 `gzip -t` 確認能完整解開，壞檔在動資料庫之前中止。
4. 呼叫 `docker-backup-mysql.sh` 備份還原前的現況（預設輸出到 `backups/mysql-pre-restore`）。
5. 解壓或直接串流到 MySQL。

執行前先停掉會寫入 MySQL 的應用服務；腳本不會替你停。

## 主要參數
- 第一個位置參數：`mysql-backup.sql.gz` 或 `mysql-backup.sql`
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`CONFIRM_RESTORE`（必須為 `yes`）、`PRE_RESTORE_BACKUP_DIR`、`SKIP_PRE_RESTORE_BACKUP=1`（略過還原前備份）

## 相關文件
- [Docker備份MySQL](./Docker備份MySQL.md)
