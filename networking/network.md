# Networking Interview Questions & Answers

---

**Q: How does DNS resolution work step by step?**

**A:**
1. **Browser/application cache**: the OS checks its own DNS cache first. If the record is found and TTL hasn't expired, resolution stops here.
2. **OS stub resolver**: the application queries the OS resolver (e.g., `/etc/resolv.conf` on Linux). The OS checks `/etc/hosts` first, then forwards to the configured recursive resolver.
3. **Recursive resolver (ISP or 8.8.8.8)**: if it has the answer cached, it returns it. Otherwise it starts iterative resolution:
   a. Queries a **root nameserver** (`.`) → gets referral to the TLD nameserver.
   b. Queries the **TLD nameserver** (e.g., `.com`) → gets referral to the authoritative nameserver.
   c. Queries the **authoritative nameserver** (e.g., Route53 for `example.com`) → gets the actual A/AAAA record.
4. The recursive resolver caches the answer (respecting the record's TTL) and returns it to the client.
5. The client caches the response and connects to the returned IP.

**DNS record types**: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (verification, SPF), NS (nameserver), SOA (zone authority), SRV (service location).

**In Kubernetes**: pods use CoreDNS as the cluster resolver. `<service>.<namespace>.svc.cluster.local` resolves via CoreDNS to the ClusterIP; the search domain list in `/etc/resolv.conf` allows short-form queries.

---

**Q: A kube-proxy update rolls out mid-match. What’s your network rollback plan to avoid packet drops?**

**A:** kube-proxy manages iptables/ipvs rules that implement Service ClusterIP routing. A bad update can break cluster-internal DNS, Service routing, and NodePort access mid-flight.

**Pre-rollout — prevention**:
- kube-proxy runs as a DaemonSet. Before updating, set a `PodDisruptionBudget` and use `maxUnavailable: 1` in the DaemonSet update strategy to limit blast radius.
- Test the new kube-proxy version in a non-prod cluster first. Validate with: `curl http://<ClusterIP>:<port>` from a test pod, `kubectl exec ... -- nslookup kubernetes.default`, and NodePort reachability.
- Take a snapshot of current iptables rules on a sample node before updating: `iptables-save > /tmp/kube-proxy-rules-before.txt`.

**Mid-rollout rollback**:
```bash
# Roll back the DaemonSet to the previous image
kubectl rollout undo daemonset/kube-proxy -n kube-system

# Monitor rollout on each node
kubectl rollout status daemonset/kube-proxy -n kube-system

# Verify kube-proxy pod is healthy on each node
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

**If iptables rules are corrupted on a node**:
```bash
# SSH to the affected node
# Flush kube-proxy-managed chains (kube-proxy will repopulate on restart)
iptables -t nat -F KUBE-SERVICES
iptables -t nat -F KUBE-NODEPORTS
systemctl restart kube-proxy   # forces rule rewrite

# Or if using ipvs mode
ipvsadm -C   # flush IPVS table; kube-proxy will repopulate
```

**For a live event (mid-match)**:
- If rollout is in progress and the first node shows problems, immediately pause:
  ```bash
  kubectl rollout pause daemonset/kube-proxy -n kube-system
  ```
- Assess: are affected nodes taking production traffic? If yes, cordon and drain them to redirect traffic to healthy nodes while you investigate.
- Rollback: `kubectl rollout undo` then `kubectl rollout resume`.
- Existing TCP connections survive kube-proxy restarts (iptables rules remain in the kernel even when kube-proxy is down; new connections may fail during the window between old rules being flushed and new rules being written).

---

**Q: NAT gateway costs double in 24 hours during a live series; no infra changes were made. What could be silently causing it?**

**A:** NAT Gateway charges per GB of data processed. Doubling with no infra changes means data volume through the NAT doubled. The culprit is almost always application or traffic behavior, not infrastructure.

**Investigation steps**:

1. **Identify which subnet/ENI the NAT traffic is from**:
   ```bash
   # Enable VPC Flow Logs if not already on, then query in CloudWatch Logs Insights or Athena
   # Filter for traffic to the NAT Gateway ENI
   fields @timestamp, srcAddr, dstAddr, bytes, action
   | filter dstAddr = "<nat-gateway-private-ip>"
   | stats sum(bytes) as total_bytes by srcAddr
   | sort total_bytes desc
   | limit 20
   ```

2. **Common causes**:
   - **Live stream egress rerouted through NAT**: if a CDN origin-pull or media packaging service lost its direct internet path and started routing through private instances → NAT Gateway. Check if CloudFront or CDN is still routing directly to origins.
   - **Logging or telemetry explosion**: an application logging verbosely to an external aggregator (Datadog, Splunk) suddenly generating 10× more logs due to a bug or debug mode left on.
   - **Software update / package pull loop**: an instance or container in a loop retrying a failed package install/update, each attempt pulling packages via NAT.
   - **Inter-AZ data transfer misidentified**: inter-AZ traffic isn’t NAT Gateway traffic, but it can cause confusion in billing. Confirm it’s actually NAT Gateway line items in Cost Explorer.
   - **S3 gateway endpoint removed or bypassed**: if the S3 VPC Gateway Endpoint was accidentally removed from a route table, S3 traffic now flows through the NAT Gateway ($0.045/GB instead of $0).
   - **New workload without VPC endpoints**: a new Lambda function or ECS task added by another team that calls AWS services (SSM, ECR, Secrets Manager) without interface endpoints — all traffic goes through NAT.
   - **DDoS / bot traffic**: compromised instance making large outbound connections. Check VPC Flow Logs for unusual destination IPs or ports.

3. **Quick checks**:
   ```bash
   # CloudWatch: NAT Gateway BytesOutToDestination metric, split by NAT GW ID
   aws cloudwatch get-metric-statistics \
     --namespace AWS/NATGateway \
     --metric-name BytesOutToDestination \
     --dimensions Name=NatGatewayId,Value=<nat-id> \
     --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z \
     --period 3600 --statistics Sum

   # Compare S3 VPC endpoint route table entries
   aws ec2 describe-route-tables --filters Name=route.origin,Values=CreateRoute \
     | jq ‘.RouteTables[].Routes[] | select(.DestinationPrefixListId != null)’
   ```

4. **Fix**:
   - Restore S3/DynamoDB Gateway Endpoints to route tables if missing.
   - Add Interface Endpoints for high-traffic AWS services.
   - Fix the logging/telemetry runaway if applicable.
   - Use AWS Cost Anomaly Detection with an alert threshold to catch this automatically next time.
