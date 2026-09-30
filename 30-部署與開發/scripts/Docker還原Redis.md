# `scripts/docker-restore-redis.sh`

## 作用
把 Redis 備份還原回 production Redis volume。

## 目的
在清掉壞資料後，快速把 volume 還原回來。

## 流程
1. 停掉 Redis。
2. 用 `busybox` 把 tar.gz 解回 volume。
3. 再把 Redis 起回來。

## 主要參數
- 第一個位置參數：`redis-backup.tar.gz`
- 透過環境變數控制：`ENV_FILE`、`COMPOSE_FILE`、`VOLUME_NAME`

## 相關文件
- [Docker備份Redis](./Docker備份Redis.md)
