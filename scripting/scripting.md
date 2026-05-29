# Scripting Interview Questions & Answers

---

**Q: Write a Bash script to check disk usage and CPU usage.**

**A:**

```bash
#!/usr/bin/env bash
# check-resources.sh — report disk and CPU usage, alert on thresholds

set -euo pipefail

DISK_THRESHOLD=80    # alert if any mount point exceeds this % used
CPU_THRESHOLD=85     # alert if CPU usage exceeds this %
CPU_SAMPLE_SECS=2    # seconds to sample CPU over

# ──────────────────────────────────────────────
# Disk usage
# ──────────────────────────────────────────────
check_disk() {
  echo "=== Disk Usage ==="
  local alert=0

  # df -h: human-readable; skip header and tmpfs/devtmpfs pseudo-filesystems
  while IFS= read -r line; do
    local used_pct mount
    used_pct=$(echo "$line" | awk '{print $5}' | tr -d '%')
    mount=$(echo "$line" | awk '{print $6}')

    printf "%-30s %s%%\n" "$mount" "$used_pct"

    if (( used_pct >= DISK_THRESHOLD )); then
      echo "  ⚠ ALERT: $mount is at ${used_pct}% (threshold: ${DISK_THRESHOLD}%)"
      alert=1
    fi
  done < <(df -h --output=source,size,used,avail,pcent,target \
              | grep -vE '^Filesystem|tmpfs|devtmpfs|udev')

  return $alert
}

# ──────────────────────────────────────────────
# CPU usage (via /proc/stat — works on Linux)
# ──────────────────────────────────────────────
check_cpu() {
  echo ""
  echo "=== CPU Usage ==="

  # Read two samples of /proc/stat separated by SAMPLE_SECS seconds
  # Line format: cpu  user nice system idle iowait irq softirq steal guest guest_nice
  read_cpu_stat() {
    awk '/^cpu / {print $2, $3, $4, $5, $6, $7, $8}' /proc/stat
  }

  local stat1 stat2
  stat1=$(read_cpu_stat)
  sleep "$CPU_SAMPLE_SECS"
  stat2=$(read_cpu_stat)

  local cpu_pct
  cpu_pct=$(awk -v s1="$stat1" -v s2="$stat2" 'BEGIN {
    split(s1, a, " "); split(s2, b, " ")
    # total ticks = user+nice+system+idle+iowait+irq+softirq
    idle1 = a[4]; total1 = 0; for (i=1;i<=7;i++) total1 += a[i]
    idle2 = b[4]; total2 = 0; for (i=1;i<=7;i++) total2 += b[i]
    diff_idle  = idle2  - idle1
    diff_total = total2 - total1
    usage = 100 * (diff_total - diff_idle) / diff_total
    printf "%.1f", usage
  }')

  echo "CPU usage (${CPU_SAMPLE_SECS}s sample): ${cpu_pct}%"

  if awk "BEGIN {exit !($cpu_pct >= $CPU_THRESHOLD)}"; then
    echo "  ⚠ ALERT: CPU at ${cpu_pct}% (threshold: ${CPU_THRESHOLD}%)"
    return 1
  fi

  return 0
}

# ──────────────────────────────────────────────
# Main
# ──────────────────────────────────────────────
main() {
  local exit_code=0

  check_disk  || exit_code=1
  check_cpu   || exit_code=1

  echo ""
  if (( exit_code == 0 )); then
    echo "✓ All checks passed."
  else
    echo "✗ One or more thresholds exceeded."
  fi

  exit $exit_code
}

main "$@"
```

**Usage**:
```bash
chmod +x check-resources.sh
./check-resources.sh

# Example output:
# === Disk Usage ===
# /                              62%
# /var/lib/docker                91%
#   ⚠ ALERT: /var/lib/docker is at 91% (threshold: 80%)
#
# === CPU Usage ===
# CPU usage (2s sample): 23.4%
#
# ✗ One or more thresholds exceeded.
```

**Key points**:
- **`/proc/stat` sampling**: CPU usage is computed from the difference between two readings — a single reading only gives cumulative ticks since boot, not current utilization.
- **`set -euo pipefail`**: exits on error, treats unset variables as errors, and catches pipe failures.
- **`df --output=`**: uses GNU `df` column selection for clean parsing. On macOS replace with `df -h | awk 'NR>1 {print $5, $9}'`.
- **Non-zero exit**: the script exits 1 if any threshold is breached, making it suitable as a cron job or CI health check gate.
- **Extend it**: add `--email` flag to send alerts via `mail`, or `POST` to a Slack webhook using `curl`.

---
