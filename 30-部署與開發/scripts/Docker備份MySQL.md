# `scripts/docker-backup-mysql.sh`

## 作用
匯出 production MySQL 備份。

## 目的
在維運或發版前保留資料回滾點。

## 流程
1. 載入 `.env.prod`。
2. 連到 compose 裡的 `mysql`。
3. 用 `mysqldump` 匯出後壓成 `.sql.gz`。

## 主要參數
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`BACKUP_DIR`、`TIMESTAMP`、`OUTPUT_FILE`

## 相關文件
- [Docker還原MySQL](./Docker還原MySQL.md)
