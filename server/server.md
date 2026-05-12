# Server Interview Questions & Answers

---

**Q: What happens when your system goes down - how do you approach it?**

**A:** Structured incident response:

1. **Detect & acknowledge**: confirm the outage (monitoring alert, user report, status page). Acknowledge in PagerDuty/Opsgenie to stop alert escalation. Post in the incident Slack channel: "Investigating [service] outage, started [time]."
2. **Assess blast radius**: who is affected? All users? A specific region? A single tenant?
3. **Mitigate first, fix later**: if there's a quick mitigation (rollback a recent deploy, failover to another region, scale up), do it immediately. Don't wait for root cause before stopping user impact.
4. **Gather signals**: check dashboards (Grafana/Datadog), logs (Kibana/Loki), recent changes (git log, deploy history), cloud provider status page.
5. **Hypothesize and test**: form a theory based on signals; validate with a targeted check.
6. **Fix and verify**: apply the fix; watch metrics normalize; verify from a user perspective.
7. **Communicate**: keep stakeholders updated every 15–30 minutes during the incident.
8. **Post-mortem**: blameless RCA within 48 hours; document timeline, root cause, contributing factors, and action items to prevent recurrence.

---

**Q: Your app teams demand custom AMIs. What's your pre-prod vetting strategy at kernel and runtime?**

**A:**
1. **Automated build pipeline**: use HashiCorp Packer to build AMIs from a base (Amazon Linux 2023, Ubuntu) with a standardized provisioning script. All customizations go through Packer — no manual AMI modifications.
2. **CIS hardening**: apply the CIS Level 1 benchmark automatically in the Packer build using Ansible roles (e.g., `dev-sec/cis-hardening`).
3. **Security scanning**: run AWS Inspector or Trivy on the built AMI; fail the pipeline if HIGH/CRITICAL kernel or package CVEs are found.
4. **Functional testing**: launch a test EC2 from the AMI; run automated tests:
   - `cloud-init` completes without error.
   - Required services start (`kubelet`, `containerd`, `ssm-agent`).
   - Kernel version meets minimum requirement.
   - SELinux/AppArmor in enforcing mode.
   - auditd is running and logging.
5. **Performance baseline**: run `sysbench` or `fio` to confirm I/O performance regressions aren't introduced.
6. **Staged rollout**: deploy to dev node group first; monitor for 24h; then staging; then production node groups using a blue-green ASG replacement.
7. **Approval gate**: security team sign-off required before production AMI promotion.

---

**Q: You deployed a sidecar logging agent. Suddenly, CPU throttling spikes. Diagnose and rollback.**

**A:**
**Diagnose**:
```bash
# Which pods are being throttled?
kubectl top pods -A --sort-by=cpu

# Check throttling metric (cgroup-level)
# On the node:
cat /sys/fs/cgroup/cpu/kubepods/pod<pod-uid>/<container-id>/cpu.stat
# Look for: nr_throttled and throttled_time

# Prometheus metric for throttling
rate(container_cpu_cfs_throttled_seconds_total{container="logging-agent"}[5m])
```

**Root cause candidates**:
- The logging agent has a very low `resources.limits.cpu`, causing CFS throttling when log volume spikes.
- The agent is doing expensive regex parsing or compression on every log line.
- The log volume itself increased (e.g., verbose debug logging turned on accidentally).

**Rollback**:
```bash
# Rollback the DaemonSet to previous version
kubectl rollout undo daemonset/logging-agent -n logging

# Verify
kubectl rollout status daemonset/logging-agent -n logging
```

**Fix**: increase `resources.limits.cpu` for the sidecar, add log sampling/filtering to reduce processing volume, or switch to async batched log shipping.

---

**Q: Systemd journal logs vanish on reboot across some AMIs. What do you check in the image build and boot sequence?**

**A:**
1. **Journal storage mode**: check `/etc/systemd/journald.conf` for `Storage=`. If `Storage=volatile` (default on some distros), journals are in `/run/log/journal` which is cleared on reboot. Fix: set `Storage=persistent` to write to `/var/log/journal`.
2. **Journal directory**: confirm `/var/log/journal/<machine-id>` directory exists and has correct permissions (`systemd-journal` group). If the directory doesn't exist, journald falls back to volatile.
3. **Packer provisioner**: check if the Packer build script deletes `/var/log/journal` as part of cleanup (common in image hardening scripts). If so, add a step to recreate it.
4. **cloud-init**: check if cloud-init or user data scripts are inadvertently clearing `/var/log`.
5. **AMI snapshot timing**: if the AMI snapshot is taken before `journald` creates the persistent dir on first boot, subsequent instances won't have it. Fix: add `mkdir -p /var/log/journal` and `systemd-tmpfiles --create` to the Packer provisioner.
6. **Machine ID**: `/etc/machine-id` must be stable (or cleared for golden AMIs and regenerated on first boot). Journal dirs are keyed by machine ID.

---

**Q: Kernel panic on a GKE node mid-deploy. How do you identify if it's infra, base image, or app-level?**

**A:**
1. **Collect crash data**: GKE nodes write kernel panic info to the serial console. Access via `gcloud compute instances get-serial-port-output <instance-name>`. Look for `Kernel panic`, `BUG:`, `Oops:`, or `Call Trace:` in the output.
2. **Identify the responsible kernel module**: the call trace shows the faulting module. If it's a driver or vendor kernel module → infrastructure issue (VM hypervisor, hardware, NVIDIA driver, etc.). If it's the container runtime (containerd/runc) → base image or runtime issue.
3. **Correlate with deploy**: did the panic happen immediately after deploying a new workload? Check if the panic correlates with a specific container/image.
   - eBPF programs (Cilium, Falco) can trigger kernel panics on buggy kernels — check if the workload uses eBPF.
   - Containers with `privileged: true` or host PID/network namespace sharing can cause kernel-level issues.
4. **GKE node logs**: `gcloud logging read "resource.type=gce_instance AND resource.labels.instance_id=<id>"` for OS-level logs before the panic.
5. **Base image**: compare the panicking node's kernel version (`uname -r`) vs healthy nodes. If the GKE node pool was recently upgraded, file a support ticket with GCP if it's a known kernel bug.
6. **App-level**: if the call trace points to a syscall made by the application (e.g., io_uring, eBPF), that's app-level. Review the application for unsafe kernel interactions.

---

**Q: How do you check network connectivity between two servers?**

**A:**
```bash
# Basic connectivity
ping <target-ip>

# Port-level connectivity
telnet <target-ip> <port>
nc -zv <target-ip> <port>              # netcat, non-interactive
curl -v http://<target-ip>:<port>     # HTTP-level check

# Trace the network path
traceroute <target-ip>                 # ICMP/UDP
tracepath <target-ip>                  # doesn't require root
mtr <target-ip>                        # continuous traceroute with loss stats

# Check if a port is listening on the target
ss -tlnp | grep <port>                 # on the target server
netstat -tlnp | grep <port>

# DNS resolution
dig <hostname> @<dns-server>
nslookup <hostname>

# Check firewall / security groups
# On Linux: iptables -L -n -v
# On AWS: check Security Group inbound/outbound rules and NACLs

# Advanced: check MTU / packet loss
ping -s 1472 <target-ip>              # test with large packets (1472 = 1500 MTU - 28 IP/ICMP headers)
```

For cross-VPC or cross-account issues in AWS: also verify VPC peering routes, Transit Gateway route tables, and Security Group rules allow the specific port/protocol.
