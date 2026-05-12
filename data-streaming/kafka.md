# Kafka / Data Streaming Interview Questions & Answers

---

**Q: Your Kafka ingestion pipeline lags by 2 minutes during a traffic surge. Producers are fine, consumers are idle. What's your debug path?**

**A:** "Producers are fine, consumers are idle" is a critical clue — the data is reaching Kafka but consumers are not processing it. This points to a problem between the broker and the consumer, not the producer side.

**Step 1 — Confirm the lag and isolate the partition**:
```bash
# Check consumer group lag per partition
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe --group my-consumer-group

# Output shows: TOPIC, PARTITION, CURRENT-OFFSET, LOG-END-OFFSET, LAG, CONSUMER-ID, HOST
# Look for: which partitions have lag, and whether CONSUMER-ID is blank (no active consumer on that partition)
```

If `CONSUMER-ID` is blank for lagging partitions → **no consumer is assigned to those partitions** — rebalance issue or consumer count < partition count.

**Step 2 — Check if consumers are actually idle (not just slow)**:
```bash
# Prometheus metrics (if Kafka consumer metrics are exported)
# kafka_consumer_fetch_manager_records_lag_max
# kafka_consumer_fetch_manager_fetch_latency_avg

# Consumer application logs — are they throwing exceptions? Stuck in a slow processing loop?
kubectl logs <consumer-pod> --tail=200 | grep -E 'ERROR|WARN|rebalance|paused'
```

**Common root causes and fixes**:

1. **Consumer group rebalancing loop**: during a surge, if consumer pods are added/removed faster than the rebalance stabilizes, consumers spend most of their time rebalancing instead of consuming.
   - Check: `kafka_consumer_coordinator_rebalance_total` metric spiking.
   - Fix: use **Static Membership** (`group.instance.id`) — prevents rebalance on brief disconnects. Tune `session.timeout.ms` (increase to 60s) and `heartbeat.interval.ms` (set to `session.timeout.ms / 3`).

2. **Consumer paused due to back-pressure**: if the downstream sink (DB, API, cache) is slow, the consumer calls `consumer.pause()` or the processing loop blocks, causing it to stop polling. Kafka's consumer will be kicked out of the group if it doesn't poll within `max.poll.interval.ms` (default 5 minutes).
   - Fix: decouple fetching from processing — use a bounded in-memory queue in the consumer; increase `max.poll.interval.ms` if processing is legitimately slow.

3. **Partition count < consumer instance count**: excess consumers sit idle (one consumer per partition is the ceiling). The surge exposed the lack of parallelism.
   - Fix: increase partition count (`kafka-topics.sh --alter --partitions 24`). Note: partition count can only increase, never decrease.

4. **Consumer fetch size too small for surge volume**: `fetch.min.bytes` and `fetch.max.bytes` defaults may cause many small fetches, not keeping pace.
   - Fix: increase `fetch.max.bytes` (default 50MB) and `max.partition.fetch.bytes`.

5. **GC pauses in consumer JVM**: under surge memory pressure, full GC pauses block the poll loop long enough for the coordinator to consider the consumer dead, triggering rebalance.
   - Check: consumer JVM GC logs (`-Xlog:gc*`).
   - Fix: tune heap size, use G1GC or ZGC, or move to a non-JVM consumer (librdkafka-based).

6. **Broker throttling the consumer**: check `kafka_server_fetch_throttle_time_avg` on the broker. If consumers are hitting byte-rate quotas, add a consumer quota or adjust the quota configuration.

**Immediate mitigation**:
- Scale out the consumer group (add more instances, up to the partition count).
- If consumers are stuck in rebalance: do a rolling restart with `group.instance.id` set to force static membership.
- Check and reset any consumer offsets that got stuck: `kafka-consumer-groups.sh --reset-offsets`.
