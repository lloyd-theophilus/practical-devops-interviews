# Linux Interview Questions & Answers

---

**Q: What's your Linux-level checklist before approving any custom AMI to production?**

**A:**
1. **OS hardening**: CIS Benchmark compliance scan (using `oscap` or AWS Inspector); disable unused services (disable `rpcbind`, `avahi-daemon`, etc.).
2. **Kernel version**: confirm the kernel is patched against known CVEs; check `uname -r`.
3. **SSH hardening**: `PermitRootLogin no`, `PasswordAuthentication no`, `AllowUsers` list; check `/etc/ssh/sshd_config`.
4. **Audit logging**: auditd installed and configured; `/var/log/audit/audit.log` persists across reboots.
5. **File system integrity**: AIDE or Tripwire baseline for critical system files.
6. **No default credentials**: verify no test users, default passwords, or embedded secrets.
7. **Systemd units**: only required services enabled on boot (`systemctl list-unit-files --state=enabled`).
8. **SELinux/AppArmor**: enforcing mode, not permissive or disabled.
9. **Package audit**: `rpm -qa` or `dpkg -l` — no unnecessary packages; all packages from approved repos.
10. **Golden image scan**: Trivy or Grype image scan for HIGH/CRITICAL CVEs; Packer + AWS Inspector for AMI-level scanning.
11. **Cloud-init idempotency**: `cloud-init` runs cleanly without errors on first and subsequent boots.
12. **FIPS mode** (if required): verify with `fips-mode-setup --check`.

---

**Q: Bash One-liner: Find all running containers using more than 500MB RSS memory on a node.**

**A:**
```bash
docker stats --no-stream --format "{{.Name}} {{.MemUsage}}" | awk '{split($2,a,"/"); val=a[1]; if(val ~ /GiB/) {gsub(/GiB/,"",val); if(val*1024 > 500) print $1, val"GiB"} else if(val ~ /MiB/) {gsub(/MiB/,"",val); if(val+0 > 500) print $1, val"MiB"}}'
```

Alternative using `/proc` (RSS in kB for each container's main process):
```bash
for cid in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' $cid)
  rss=$(awk '/^VmRSS/{print $2}' /proc/$pid/status 2>/dev/null)
  name=$(docker inspect -f '{{.Name}}' $cid)
  [ "$rss" -gt 512000 ] 2>/dev/null && echo "$name RSS: $((rss/1024))MB"
done
```

---

**Q: How do you check running processes in Linux?**

**A:**
```bash
ps aux                      # all processes, user-readable format
ps -ef                      # full format with parent PID
ps auxf                     # tree view
top                         # interactive, real-time
htop                        # interactive with color, easier navigation
pgrep -a nginx              # find by name, show command line
pidof nginx                 # PID(s) of a named process
systemctl status nginx      # for systemd-managed services
```

---

**Q: What's the difference between top, htop, and ps?**

**A:**

| Tool | Nature | Key Features |
|---|---|---|
| `ps` | Snapshot | Non-interactive, scriptable, point-in-time view; supports wide format options |
| `top` | Real-time, interactive | Built-in, sorts by CPU/memory, load average, basic kill support |
| `htop` | Real-time, interactive | Color UI, mouse support, tree view, horizontal scrolling, easier to read; requires installation |

Use `ps` in scripts and pipelines. Use `top`/`htop` for interactive triage. `htop` is significantly more user-friendly for interactive use.

---

**Q: How to schedule a cron job every 15 minutes?**

**A:**
```bash
crontab -e
```
Add:
```
*/15 * * * * /path/to/script.sh >> /var/log/myjob.log 2>&1
```

The cron expression `*/15 * * * *` means: every 15th minute of every hour, every day.

For systemd-based systems, prefer a systemd timer for better logging, dependency management, and reliability:
```ini
# /etc/systemd/system/myjob.timer
[Timer]
OnCalendar=*:0/15
Persistent=true
[Install]
WantedBy=timers.target
```

---

**Q: What is the difference between hard link and soft link?**

**A:**

| | Hard Link | Soft Link (Symbolic Link) |
|---|---|---|
| Points to | Inode directly | Path of the target file |
| Works across filesystems | No | Yes |
| Works across partitions | No | Yes |
| Target deleted | File data remains accessible | Link becomes broken (dangling) |
| Directories | Not allowed (normally) | Allowed |
| `ls -l` indicator | No special indicator | `->` points to target |

```bash
ln file.txt hardlink.txt         # hard link
ln -s /path/to/file.txt symlink  # symbolic link
```

---

**Q: How to find which process is using high memory?**

**A:**
```bash
# Sort by memory usage descending
ps aux --sort=-%mem | head -20

# Real-time top memory consumers
top -o %MEM

# Detailed memory map for a specific PID
cat /proc/<pid>/status | grep -E 'VmRSS|VmSwap|VmPeak'
pmap -x <pid>

# smem for proportional set size (more accurate for shared memory)
smem -r -s pss | head -20
```

---

**Q: What command would you use to find files larger than 100MB?**

**A:**
```bash
find /path -type f -size +100M

# With human-readable sizes
find /path -type f -size +100M -exec ls -lh {} \;

# Sort by size
find /path -type f -size +100M -printf '%s %p\n' | sort -rn | head -20

# Using du for directories
du -ah /path | sort -rh | head -20
```

---

**Q: Write a Python script to backup all files older than 30 days from a directory.**

**A:**
```python
#!/usr/bin/env python3
import os
import shutil
import time
from pathlib import Path
from datetime import datetime

SOURCE_DIR = "/data/source"
BACKUP_DIR = "/data/backup"
AGE_DAYS = 30

def backup_old_files(source: str, backup: str, age_days: int) -> None:
    threshold = time.time() - age_days * 86400
    backup_path = Path(backup)
    backup_path.mkdir(parents=True, exist_ok=True)

    for item in Path(source).iterdir():
        if item.is_file() and item.stat().st_mtime < threshold:
            dest = backup_path / item.name
            shutil.move(str(item), str(dest))
            print(f"Moved: {item} -> {dest}")

if __name__ == "__main__":
    backup_old_files(SOURCE_DIR, BACKUP_DIR, AGE_DAYS)
```

Run as a cron job:
```
0 2 * * * /usr/bin/python3 /opt/scripts/backup_old_files.py >> /var/log/backup.log 2>&1
```
