# `scripts/docker-restore-mysql.sh`

## 作用
把 MySQL 備份還原回 compose 裡的 `mysql`。

## 目的
事故後快速把資料恢復回去。

## 流程
1. 讀 `.env.prod`。
2. 檢查輸入備份檔存在。
3. 解壓或直接串流到 MySQL。

## 主要參數
- 第一個位置參數：`mysql-backup.sql.gz` 或 `mysql-backup.sql`
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`

## 相關文件
- [Docker備份MySQL](./Docker備份MySQL.md)
