---
title:  "Redis Cluster Config"
---

{% include toc icon="cog" title="Redis Cluster Config" %}

```bash
bind 10.200.1.70 127.0.0.1
protected-mode yes
port 7000
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis_7000.pid
loglevel notice
logfile "/home/weihu/redis-4.0.10/logs/Redis_7000.log"
databases 16
always-show-logo yes
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump_7000.rdb
dir /home/weihu/redis-4.0.10/data
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
slave-lazy-flush no
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble no
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
maxmemory 26843545600
requirepass tccuser
masterauth tccuser
aof-rewrite-incremental-fsync yes
cluster-enabled  yes
cluster-config-file  nodes_7000.conf
cluster-node-timeout  15000
rename-command CONFIG ""
rename-command flushall ""
rename-command flushdb ""
rename-command shutdown shotdown_dvwa

```

