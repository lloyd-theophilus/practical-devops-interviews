# AWS Database Interview Questions & Answers

---

**Q: A production RDS database is running out of storage. What immediate actions would you take to avoid downtime?**

**A:**
**Immediate (within minutes)**:
1. **Enable RDS Storage Autoscaling** (if not already on): this is the zero-downtime fix. RDS automatically expands storage when free space drops below the threshold. Enable it immediately:
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier prod-db \
     --max-allocated-storage 1000 \    # max GB to auto-scale to
     --apply-immediately
   ```
2. **Manually increase allocated storage** (non-disruptive for most engines — MySQL, PostgreSQL, MariaDB):
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier prod-db \
     --allocated-storage 500 \
     --apply-immediately
   ```
   Storage scaling is online (no downtime) for most RDS engine types with Multi-AZ.

**Identify root cause**:
- **Bloated tables**: `SELECT table_name, table_rows, data_length + index_length FROM information_schema.TABLES ORDER BY data_length + index_length DESC LIMIT 20;`
- **Transaction logs**: check if binary logs (`SHOW BINARY LOGS;`) are accumulating because `expire_logs_days` is not set.
- **Unvacuumed tables** (PostgreSQL): `SELECT * FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;` — run `VACUUM ANALYZE`.
- **Large temp files**: check for unoptimized queries generating massive temp tables.

**Prevention**: set CloudWatch alarm on `FreeStorageSpace < 20%`; enable Storage Autoscaling from day one; schedule periodic VACUUM/OPTIMIZE TABLE jobs.

---

**Q: How do you establish a connection with databases in your deployments or infrastructure setup?**

**A:**
- **Connection string**: store the RDS endpoint, port, DB name, username, and password in AWS Secrets Manager. Applications retrieve them at startup:
  ```python
  import boto3, json
  secret = json.loads(boto3.client('secretsmanager').get_secret_value(SecretId='prod/rds/mydb')['SecretString'])
  conn = psycopg2.connect(host=secret['host'], dbname=secret['dbname'], user=secret['username'], password=secret['password'])
  ```
- **IAM authentication** (passwordless): for EC2/Lambda with an IAM role, generate a temporary auth token:
  ```bash
  aws rds generate-db-auth-token \
    --hostname mydb.cluster-xyz.us-east-1.rds.amazonaws.com \
    --port 5432 \
    --username myapp_user
  ```
  No static password needed; token expires in 15 minutes.
- **RDS Proxy**: for Lambda or high-concurrency applications — RDS Proxy pools connections so Lambda doesn't exhaust the database's `max_connections` limit. Connection string points to the Proxy endpoint, not directly to RDS.
- **Security**: RDS is always in a private subnet. EC2/Lambda connects via Security Group rules (allow app SG → RDS SG on port 5432/3306). Never expose RDS publicly.
- **Kubernetes (EKS)**: use the Secrets Store CSI Driver to mount RDS credentials from Secrets Manager as files/env vars in pods.

---

**Q: How do you reduce RDS cost without downtime?**

**A:**
1. **Right-size the instance**: use CloudWatch metrics `CPUUtilization`, `DatabaseConnections`, `FreeableMemory` + Performance Insights to find over-provisioned instances. Downsize during a maintenance window (brief failover with Multi-AZ, typically < 60 seconds).

2. **Reserved Instances**: purchase 1-year or 3-year RDS Reserved Instances for stable workloads (up to 69% savings over on-demand).

3. **Aurora Serverless v2** (for variable workloads): scales ACUs (Aurora Capacity Units) up and down automatically; ideal for dev/test or spiky production workloads. No charge when paused (v2 can scale to 0.5 ACU minimum).

4. **Stop non-prod RDS instances**: RDS instances can be stopped for up to 7 days (automatically restart after 7 days). Schedule stop/start with EventBridge + Lambda for dev/QA databases.

5. **Switch from Multi-AZ to Single-AZ** for non-critical environments (saves ~50%). Never do this for production.

6. **Storage optimization**:
   - Use gp3 storage instead of gp2 (20% cheaper per GB).
   - Enable storage auto-scaling to avoid over-provisioning.
   - Delete old manual snapshots; retain only automated backups for the required retention period.

7. **Move cold data**: use DMS to archive old data to S3 + Athena for historical queries; reduce active DB size.

8. **Use Aurora vs RDS**: Aurora can be more cost-efficient at scale due to its storage model (pay per GB used, not provisioned); Aurora I/O-Optimized pricing eliminates per-I/O charges for I/O-heavy workloads.

9. **Read replicas**: scale reads with Read Replicas instead of vertically scaling the primary (which is more expensive per unit of read throughput).

---

**Q: During a Cloud SQL failover, half the read replicas stall in “catch-up” state. What’s happening under the hood?**

**A:** This is a replication lag problem that surfaces during failover. Here’s what’s happening and why it affects only some replicas:

**Under the hood**:
- Cloud SQL uses asynchronous replication (PostgreSQL streaming replication or MySQL binlog replication depending on the engine).
- During a failover, the **primary** promotes the **standby** (which uses synchronous replication and is always in sync) to become the new primary.
- **Read replicas** use **asynchronous replication** — they lag behind the primary by design. When the primary fails over, read replicas must now catch up to the new primary’s WAL (Write-Ahead Log) position.
- Replicas that were close to the old primary’s LSN catch up quickly. Replicas that had accumulated lag (network partition, heavy read load slowing apply, I/O bottleneck on the replica) are stuck in catch-up.

**Why only half?**
- Replicas in different zones/regions have different network latency to the primary → different replication lag at the time of failover.
- Replicas under heavy read load may have had their WAL apply worker starved (competing for I/O with read queries).
- Some replicas may have had connections paused by the GCP infrastructure during the failover event.

**Diagnosis**:
```sql
-- On each replica (PostgreSQL)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Check WAL receiver status
SELECT * FROM pg_stat_wal_receiver;

-- Check how far behind from primary
SELECT pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS receive_vs_replay_bytes;
```

In Google Cloud Monitoring: check `cloudsql.googleapis.com/database/replication/replica_lag` metric per instance.

**Remediation**:
- Wait — catch-up is automatic; once replication resumes from the new primary, replicas will drain the lag.
- For replicas with very large lag: if your application can’t tolerate the lag, temporarily redirect read traffic away from the lagging replicas using weighted routing in your connection pooler (PgBouncer, ProxySQL) or Cloud SQL Auth Proxy.
- **Prevention**:
  - Set `max_replication_slots` and `wal_keep_size` appropriately so the primary retains WAL for slow replicas.
  - Monitor replica lag with an alert threshold (e.g., > 30 seconds triggers a page).
  - Avoid running analytics queries directly on replicas used for operational reads — use a dedicated analytics replica.
  - Consider using Cloud SQL’s **cascaded replication** to reduce load on the primary’s replication stream.

