# `scripts/docker-backup-mysql.sh`

## 作用
匯出 production MySQL 備份。

## 目的
在維運或發版前保留資料回滾點。

## 流程
1. 以 `scripts/lib/envfile.sh` 的 `env_get` 逐鍵讀 `.env.prod`（compose dotenv 規則，不 `source`；`MYSQL_DSN` 不加引號也不會讓 bash 語法錯誤）。
2. 連到 compose 裡的 `mysql`。
3. 用 `mysqldump` 匯出後壓成 `.sql.gz`。
4. 只保留最近 `BACKUP_KEEP` 份 `mysql-*.sql.gz`（預設 20，`0` 表示不清理）；CD 部署停服後多備份的 `-final` 檔也在內。

## 主要參數
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`BACKUP_DIR`、`TIMESTAMP`、`OUTPUT_FILE`、`BACKUP_KEEP`（須為非負整數，否則在備份前以 exit 2 失敗，不會等備份完才發現輪替失效）

## 相關文件
- [Docker還原MySQL](./Docker還原MySQL.md)
