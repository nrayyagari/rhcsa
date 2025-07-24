# Chapter 11: Working with systemd - Q&A

## systemd as Standard Init System

### Q1: Is systemd the standard for managing services on Linux?
**A:** Yes, systemd has become the **de facto standard** init system and service manager on most major Linux distributions since around 2014-2015.

**Adoption across distributions:**
- **RHEL/CentOS 7+**, Fedora, Ubuntu 15.04+, Debian 8+, SUSE, Arch Linux
- **Notable holdouts**: Some minimal/embedded distros, Gentoo (optional), Alpine Linux (uses OpenRC)

**Why it became dominant:**
- **Dependency management** - services start in correct order
- **Parallel startup** - faster boot times
- **Unified logging** - centralized via journalctl
- **Socket activation** - services start on-demand
- **Cgroup integration** - resource management
- **Standardized configuration** - consistent across distros

### Q2: How does systemd relate to containers and minimal environments?
**A:** **Containers should be minimal** and typically run **single processes**, which changes the systemd picture significantly.

**Container Philosophy vs systemd:**
- **One process per container** - violates systemd's multi-service management purpose
- **Minimal base images** - systemd adds unnecessary bloat (~240MB+ overhead)
- **Process isolation** - containers provide isolation, not systemd
- **Orchestrator handles lifecycle** - Kubernetes/Docker manage what systemd would

**What containers use instead:**
- **tini** - tiny init for zombie reaping (Docker's default)
- **dumb-init** - simple process supervisor
- **s6-overlay** - lightweight process supervision
- **No init at all** - direct process execution (most common)

**Enterprise reality**: Containers made systemd largely irrelevant in cloud-native architectures. The orchestrator became the new "init system" for distributed applications.

---

## systemd Unit File Types

### Q3: What are the most popular systemd unit file types?
**A:** The **four most commonly used unit types** in production environments:

1. **.service** - Application/daemon management (Most common ~80%)
2. **.timer** - Cron replacement (better) (~15%)
3. **.target** - Grouping/runlevels (~3%)
4. **.mount** - Filesystem management (~2%)

**Usage Frequency in Production:**
- **Service** - Every daemon needs one
- **Timer** - Backup jobs, cleanup tasks, monitoring
- **Target** - System state management
- **Mount** - Network shares, external storage

### Q4: What is a .target unit and what's its main purpose?
**A:** **Yes, .target is a systemd unit** - represents system states/milestones:
- `multi-user.target` = "system ready for multiple users" (like old runlevel 3)
- `network-online.target` = "network fully configured"
- `graphical.target` = "GUI ready" (like old runlevel 5)

They **group related services** and mark **system readiness milestones**.

---

## Unit File Locations and Hierarchy

### Q5: Why are service unit files in /usr/lib/systemd/system/ rather than /etc/systemd/system/?
**A:** This follows **systemd's hierarchy** for unit files, which has specific precedence and purposes:

**File Location Hierarchy:**

**/usr/lib/systemd/system/** (Package-provided)
- **Vendor/package defaults** - installed by package managers
- **Never edit directly** - gets overwritten on package updates
- **System-wide defaults** - what the distro/package maintainer provides
- **Lower priority** in systemd's search order

**/etc/systemd/system/** (Administrator customizations)
- **Local administrator overrides** - your custom configurations
- **Survives package updates** - never touched by package managers
- **Higher priority** - overrides files in /usr/lib/
- **Where you should make changes**

### Q6: How does systemd precedence work between /usr/lib and /etc directories?
**A:** **Exactly!** `/etc` always wins - whether complete replacement or partial override.

**Complete Override:**
If `/etc/systemd/system/nginx.service` exists:
- **systemd ignores** `/usr/lib/systemd/system/nginx.service` entirely
- Uses **only** the `/etc` version
- `/usr/lib` version becomes irrelevant

**Partial Override (Drop-ins):**
If `/etc/systemd/system/nginx.service.d/override.conf` exists:
- **Merges** with `/usr/lib/systemd/system/nginx.service`
- `/etc` settings **override** conflicting `/usr/lib` settings  
- Non-conflicting settings from both are combined

**Key insight**: `/etc` always wins - whether complete replacement or partial override. Package updates can't break your customizations because they only touch `/usr/lib`.

---

## nginx.service Analysis

### Q7: What do the key directives in nginx.service mean?
**A:** Let me break down each line of the actual nginx.service file:

**[Unit] Section - Service Metadata**
- `Description=The nginx HTTP and reverse proxy server` - Human-readable description
- `After=network-online.target remote-fs.target nss-lookup.target` - **Ordering dependency** - start nginx AFTER these targets
- `Wants=network-online.target` - **Weak dependency** - try to start network-online.target if not running

**[Service] Section - Process Management**
- `Type=forking` - Nginx **forks into background** (daemonizes)
- `PIDFile=/run/nginx.pid` - **Process ID tracking** - systemd knows which process is the main nginx
- `ExecStartPre=/usr/bin/rm -f /run/nginx.pid` - **Pre-start cleanup** - remove stale PID file
- `ExecStartPre=/usr/sbin/nginx -t` - **Configuration validation** before starting
- `ExecStart=/usr/sbin/nginx` - **Main start command**
- `ExecReload=/usr/sbin/nginx -s reload` - **Graceful reload** triggered by `systemctl reload nginx`
- `KillSignal=SIGQUIT` - **Graceful shutdown signal** instead of default SIGTERM
- `TimeoutStopSec=5` - Wait **5 seconds** for graceful shutdown before SIGKILL
- `KillMode=mixed` - Send signal to **main process first**, then all remaining processes
- `PrivateTmp=true` - **Security hardening** - nginx gets private /tmp directory

**[Install] Section - Boot Behavior**
- `WantedBy=multi-user.target` - **Auto-start at boot** when system reaches multi-user mode

### Q8: What's the difference between After= and Wants= directives?
**A:** Different purposes:

- **After=** - **ORDERING ONLY** - "start me after these units"
- **Wants=** - **DEPENDENCY** - "please start these units if not already running"

Example: nginx says "Start network-online.target if it's not running (Wants) but wait until it's ready before starting me (After)"

---

## Process Forking and Daemonization

### Q9: What does "forking into background" mean and how does it work?
**A:** **Traditional daemon behavior with actual nginx example:**

**Current Running State:**
```
PID 23402: nginx: master process  <- This is the MAIN nginx daemon
├── PID 23403: nginx: worker process  <- Child of master
├── PID 23404: nginx: worker process  <- Child of master  
├── PID 23405: nginx: worker process  <- Child of master
└── PID 23406: nginx: worker process  <- Child of master
```

**The "forking" happens in TWO stages:**

**Stage 1: Daemonization (startup only)**
```bash
# When nginx starts initially:
# Step 1: /usr/sbin/nginx runs (temporary launcher PID)
# Step 2: This process forks and creates master process (PID 23402)  
# Step 3: Launcher process exits
# Step 4: Master process (23402) becomes the daemon
```

**Stage 2: Worker creation (ongoing)**  
```bash
master_process → worker_processes (master stays running)
```

**Key Insight:** The master process continues running and worker processes show master as PPID. The initial "forking and exit" only happens during daemonization to detach from the terminal.

### Q10: What is the purpose of the PID file (/run/nginx.pid)?
**A:** **/run/nginx.pid contains the Process ID:**

Example: `cat /run/nginx.pid` outputs `23402`

**systemd reads this file to know:**
- "The main nginx process is PID 23402"
- "To stop nginx, send signal to PID 23402"  
- "To check if nginx is running, check if PID 23402 exists"

**Why needed?** After forking, systemd lost track of which process is the "real" nginx. PID file tells systemd "this is the one to manage."

---

## Service Control and Termination

### Q11: How should you terminate nginx - systemctl or kill command?
**A:** **You have options:**

```bash
# Method 1: systemd way (RECOMMENDED)
systemctl stop nginx
# systemd sends SIGQUIT (graceful), waits 5s, then SIGKILL if needed

# Method 2: Manual kill (works but not recommended)
kill -15 $(cat /run/nginx.pid)  # SIGTERM
kill -3 $(cat /run/nginx.pid)   # SIGQUIT (nginx prefers this)

# Method 3: Direct PID (if you know it)
kill -15 1235
```

**Why use systemctl?** It follows the service's defined shutdown procedure (graceful signals, timeouts, cleanup).

### Q12: What's the difference between systemctl enable and systemctl start?
**A:** **Different purposes:**

```bash
# enable = "start automatically at boot"
systemctl enable nginx
# Creates symlink: /etc/systemd/system/multi-user.target.wants/nginx.service

# start = "start right now"  
systemctl start nginx

# Common pattern:
systemctl enable --now nginx  # enable + start in one command
```

**Key difference:**
- `enable` = **persistent** across reboots
- `start` = **temporary** until reboot or manual stop

**Enterprise reality**: You typically want both - `enable` for production services that should survive reboots, `start` for immediate operation.

---

## systemd vs Container Orchestration

### Q13: How does container orchestration replace systemd functionality?
**A:** **Cloud Native Reality:**

**What systemd traditionally managed:**
- **Service management** - starting/stopping/monitoring services
- **System logging** - journald
- **Network management** - networkd
- **Dependency management** - service ordering

**What replaces systemd in containers:**
- **Kubernetes** handles what systemd does (service management, logging, networking)
- **Docker Compose** orchestrates multi-service applications
- **Service mesh** (Istio) handles inter-service communication
- **Observability stack** replaces systemd logging

**Best Practice Example:**
```dockerfile
# Good - single process
FROM alpine:3.18
COPY app /usr/local/bin/
CMD ["app"]

# Bad - systemd overhead
FROM ubuntu:22.04
RUN apt-get install systemd
CMD ["/bin/systemd"]
```

**Bottom line**: Containers made systemd largely irrelevant in cloud-native architectures. The orchestrator became the new "init system" for distributed applications.

---

## Practical systemd Commands

### Q14: What are the essential systemd commands for daily administration?
**A:** **Service management:**
```bash
systemctl start service
systemctl stop service
systemctl restart service
systemctl reload service
systemctl enable service
systemctl disable service
systemctl status service
```

**System information:**
```bash
systemctl list-units                    # All active units
systemctl list-units --type=service     # Services only
systemctl list-units --failed           # Failed units
systemctl list-unit-files               # All unit files
```

**Logs and debugging:**
```bash
journalctl -u service                   # Service logs
journalctl -f                          # Follow logs real-time
journalctl -xe                         # Show recent errors
systemd-analyze                        # Boot performance
systemd-analyze blame                  # Boot time by service
```

### Q15: How do you verify what's actually running after making changes?
**A:** **Verification commands:**
```bash
systemctl cat nginx    # Shows effective merged configuration
systemctl show nginx   # Shows all active properties
systemctl status nginx # Current status and recent logs
ps aux | grep nginx    # Actual running processes
```

**Real-world workflow:**
```bash
# 1. Make changes
sudo systemctl edit nginx

# 2. Reload systemd configuration
sudo systemctl daemon-reload

# 3. Apply changes
sudo systemctl restart nginx

# 4. Verify
systemctl status nginx
systemctl cat nginx
```

---

## Enterprise Best Practices

### Q16: What are systemd best practices for production environments?
**A:** **Configuration management:**
- **Always use /etc/systemd/system/** for customizations
- **Use drop-in files** (`systemctl edit`) for partial changes
- **Version control** unit files like any configuration
- **Test changes** in non-production first

**Service design:**
- **Graceful shutdown** - handle SIGTERM properly
- **Health checks** - use systemd watchdog or external monitoring
- **Logging** - log to stdout/stderr, let systemd capture
- **Security** - use systemd security features (PrivateTmp, User=, etc.)

**Monitoring and maintenance:**
```bash
# Regular health checks
systemctl list-units --failed
journalctl --since="1 hour ago" --priority=err

# Performance monitoring
systemd-analyze critical-chain
systemd-analyze plot > bootchart.svg
```

**Enterprise reality**: Most sysadmins spend 90% of their time with .service files, occasionally create .timer files to replace cron, and rarely touch .target or .mount unless doing advanced system configuration.

---

## Real-World Scenarios

### Q17: How do you troubleshoot a service that won't start?
**A:** **Systematic troubleshooting approach:**

```bash
# 1. Check service status
systemctl status nginx

# 2. Check recent logs
journalctl -u nginx -f

# 3. Check configuration syntax
nginx -t  # Service-specific test

# 4. Check dependencies
systemctl list-dependencies nginx

# 5. Check for port conflicts
ss -tlnp | grep :80

# 6. Check file permissions
ls -la /usr/sbin/nginx
ls -la /etc/nginx/

# 7. Manual test
sudo -u nginx /usr/sbin/nginx -t
```

### Q18: How do you migrate from cron to systemd timers?
**A:** **Migration example:**

**Old cron job:**
```bash
# /etc/crontab
0 2 * * * root /usr/local/bin/backup.sh
```

**New systemd timer:**

**backup.service:**
```ini
[Unit]
Description=Daily backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=root
```

**backup.timer:**
```ini
[Unit]
Description=Daily backup timer
Requires=backup.service

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

**Enable and start:**
```bash
systemctl enable --now backup.timer
systemctl list-timers  # Verify
```

**Why better than cron?**
- **Logging** - integrated with journalctl
- **Dependencies** - can depend on other services
- **Resource management** - cgroup integration
- **Flexibility** - more scheduling options

---

## References

- systemd official documentation: systemd.io
- Red Hat systemd documentation
- nginx systemd integration guide
- Container best practices for systemd alternatives