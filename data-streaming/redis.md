**Q: Sudden tail latency on a Redis-based stream session store during a Champions League match — how do you find & fix the bottleneck?**

**A:**

**1. Diagnose in real time**

```bash
redis-cli --latency                        # live latency samples
redis-cli --latency-history -i 1           # per-second history
redis-cli SLOWLOG GET 25                   # last 25 slow commands
redis-cli INFO stats                       # rejected_connections, blocked_clients
redis-cli INFO memory                      # used_memory vs maxmemory
redis-cli LATENCY LATEST                   # requires latency monitoring enabled
```

Enable latency monitoring if not already on:

```bash
redis-cli CONFIG SET latency-monitor-threshold 10
```

**2. Common root causes and fixes**

| Root Cause | Symptoms | Fix |
|---|---|---|
| `BGSAVE` / `BGREWRITEAOF` fork | Spike every few minutes, high `fork_avg_cpumsec` | Disable RDB (`save ""`), switch AOF to `appendfsync everysec`, schedule persistence off-peak |
| Hot key (single session shard saturated) | One key dominating `MONITOR` output | Shard sessions by key prefix across Redis Cluster; add a local in-process cache for read-heavy keys |
| Memory near `maxmemory` triggering eviction | `evicted_keys` rising in `INFO stats` | Scale memory, or switch eviction policy to `allkeys-lru` and move cold sessions to a cheaper store |
| Large stream backlog / slow `XREAD` | Slow commands show `XREAD COUNT` on huge streams | Trim streams at write time: `XADD mystream MAXLEN ~ 10000 * field val` |
| Consumer group lag | `XPENDING` shows growing PEL | Add consumer instances; increase `COUNT` per `XREADGROUP` call; auto-claim stalled messages with `XAUTOCLAIM` |
| Network saturation | High `instantaneous_input_kbps` | Enable pipelining; batch reads; move to Redis 7 + RESP3 for better multiplexing |
| Blocking commands (`KEYS *`, `LRANGE` on huge lists) | Single-thread queue stalled | Replace `KEYS` with `SCAN`; cap list/set sizes; use bounded data structures |

**3. Preventive measures for traffic spikes**

- Pre-warm sessions before kick-off so the first wave of reads hits warm keys.
- Use Redis Cluster to horizontally scale — distribute session keys across shards.
- Set `tcp-backlog 511` and tune OS `somaxconn` / `net.core.somaxconn` to handle connection bursts.
- Monitor `connected_clients` and alert before hitting `maxclients`.

---
