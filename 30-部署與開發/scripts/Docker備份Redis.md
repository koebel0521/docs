# `scripts/docker-backup-redis.sh`

## 作用
匯出 production Redis volume。

## 目的
替 Redis 資料保留可回復的備份檔。

## 流程
1. 決定備份目錄與輸出檔名。
2. 用臨時 `busybox` 容器把 volume 打成 `.tar.gz`。

## 主要參數
- 透過環境變數控制：`BACKUP_DIR`、`TIMESTAMP`、`OUTPUT_FILE`、`VOLUME_NAME`

## 相關文件
- [Docker還原Redis](./Docker還原Redis.md)
