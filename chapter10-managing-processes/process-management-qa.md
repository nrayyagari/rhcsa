# Chapter 10: Managing Processes - Q&A

## Process Types

### Q1: What are the three main types of processes in Linux?
**A:** The three types are:
- **Shell jobs** - User-initiated processes tied to terminal sessions
- **Daemons** - Long-running background services detached from terminals
- **Kernel threads** - Processes running in kernel space for system operations

### Q2: How can you identify each process type?
**A:** 
- **Shell jobs**: Have TTY assigned (pts/0, tty1, etc.)
- **Daemons**: Show "?" in TTY column, PPID usually 1, user space memory
- **Kernel threads**: Names in brackets [kthreadd], zero memory (VSZ=0, RSS=0)

### Q3: What command shows the process hierarchy?
**A:** `pstree -p` shows the complete process tree with PIDs.

### Q4: Why do kernel threads have zero memory usage?
**A:** Kernel threads run entirely in kernel space and don't have virtual memory like user space processes. They share the kernel's memory space.

### Q5: What happens to shell jobs when you close the terminal?
**A:** Shell jobs receive SIGTERM and typically die when the terminal closes, unless protected with `nohup` or `disown`.

## Daemons and System Services

### Q6: What is the daemonization process?
**A:** Daemonization involves:
1. `fork()` and parent exits (becomes child of init)
2. `setsid()` - become session leader
3. `fork()` again (prevent controlling terminal)
4. `chdir("/")` - change to root directory
5. Close all file descriptors
6. Redirect stdin/stdout/stderr to `/dev/null`

### Q7: How do systemd services differ from traditional daemons?
**A:** systemd manages services with:
- Dependency management
- Parallel startup
- Unified logging (journald)
- Socket-based activation
- Resource management via cgroups

### Q8: What are some common systemd services?
**A:** Key services include:
- `systemd-resolved` - DNS resolution
- `systemd-networkd` - Network management
- `systemd-journald` - Centralized logging
- `systemd-logind` - User session management
- `systemd-udevd` - Device management

## Job Control

### Q9: When should you use `bg` and `fg` commands?
**A:** 
- **Use `bg`**: When you forgot to add `&`, need terminal back while task continues
- **Use `fg`**: When you need to interact with backgrounded process, monitor output, or stop it properly

### Q10: What are the job control key combinations?
**A:**
- `Ctrl+Z` - Suspend current foreground job
- `Ctrl+C` - Terminate current foreground job
- `&` - Start command in background
- `jobs` - List current jobs

### Q11: How do you reference jobs in commands?
**A:**
- `%1` - Job number 1
- `%+` or `%%` - Current job (most recent)
- `%-` - Previous job
- `%ping` - Job with command starting "ping"

### Q12: What are the different job states?
**A:**
- **Running** - Executing in background `[1]+ Running`
- **Stopped** - Suspended with Ctrl+Z `[1]+ Stopped`
- **Done** - Completed successfully `[1]+ Done`

### Q13: How can you track progress of background jobs?
**A:** Methods include:
- `tail -f logfile` - Monitor output files
- `ps aux | grep process` - Check if still running
- `top -p PID` - Monitor resource usage
- `kill -0 PID` - Test if process exists
- `wc -l logfile` - Count progress by lines

### Q14: What's the difference between job control and process control?
**A:**
- **Job control**: Shell's view using `jobs`, `fg`, `bg`, `kill %1`
- **Process control**: System view using `ps`, `kill PID`, `pgrep`

### Q15: How do you make a process survive shell exit?
**A:** Options include:
- `nohup command &` - Ignore hangup signal
- `disown %1` - Remove job from shell's job table
- `screen` or `tmux` - Persistent terminal sessions
- Convert to systemd service for permanent services

## Practical Commands

### Q16: What commands identify running processes?
**A:**
```bash
ps aux                    # All processes
ps axo pid,ppid,tty,comm  # Custom format
top / htop                # Real-time view
systemctl list-units      # systemd services
```

### Q17: How do you find kernel threads specifically?
**A:**
```bash
ps aux | grep "\[.*\]"    # Processes with brackets
ps -ef | awk '$3==2'      # Children of kthreadd (PID 2)
```

### Q18: What's the complete job control workflow?
**A:**
```bash
command                   # Start foreground
Ctrl+Z                   # Suspend
jobs                     # List jobs
bg %1                    # Resume in background
fg %1                    # Bring to foreground
kill %1                  # Terminate job
```

## Troubleshooting

### Q19: How do you debug a hung process?
**A:**
1. Check process state with `ps aux`
2. Look for 'D' state (uninterruptible sleep)
3. Check system load and I/O wait
4. Use `strace -p PID` to see system calls
5. Check logs in `/var/log/` or `journalctl`

### Q20: What if `jobs` command shows nothing but processes are still running?
**A:** The process may have been:
- Started in a different shell session
- Disowned from the current shell
- Started as a daemon
Use `ps aux` to find all processes regardless of shell association.

## Process Information with ps Command

### Q21: What does the basic `ps` command show without any options?
**A:** Shows processes running in the current terminal session only with columns:
- **PID**: Process ID (unique identifier)
- **TTY**: Terminal associated with the process
- **TIME**: CPU time used (not total runtime)
- **CMD**: Command that started the process

### Q22: What are the different styles of ps command options?
**A:** Three styles:
- **Unix-style**: With dash (e.g., `ps -aux`)
- **BSD-style**: No dash (e.g., `ps aux`)
- **GNU-style**: Double dash with words (e.g., `ps --sort`)

### Q23: What do common process status codes mean?
**A:** Key status codes in STAT column:
- **S**: Interruptible sleep (waiting for event)
- **D**: Uninterruptible sleep (usually I/O)
- **R**: Running or runnable
- **T**: Stopped (backgrounded or traced)
- **s**: Process leader

### Q24: How do you find the parent process ID (PPID) of a running process?
**A:** Multiple methods:
- Check `/proc/PID/status` file - shows `PPid: parent_id`
- Use `ps -o pid,ppid,comm -p PID`
- Use `pstree -p -s PID` to see full ancestry chain

### Q25: What's the difference between `pstree -p PID` and `pstree -p -s PID`?
**A:** 
- `pstree -p PID` shows only the process and its children
- `pstree -p -s PID` shows the full ancestry chain (parents and children)

### Q26: Can you kill kernel processes? Why or why not?
**A:** No, kernel processes (shown in square brackets like `[kthreadd]`) cannot be killed because:
- They run in kernel space with protection
- They handle essential system functions (memory, I/O, scheduling)
- The kernel ignores all signals sent to them
- Killing them would crash the system

### Q27: What happens if you kill PID 1?
**A:** Killing PID 1 (init/systemd) causes:
- Immediate kernel panic
- System becomes unresponsive
- All processes die
- Requires hard reboot

### Q28: What are the most useful ps command variations?
**A:** Essential commands:
- `ps aux` - All processes with detailed info (CPU%, MEM%, START time)
- `ps x` - All user processes regardless of terminal
- `ps aux --sort=-%cpu` - Sort by CPU usage (highest first)
- `ps aux --sort=-%mem` - Sort by memory usage (highest first)
- `ps -ejH` or `ps axjf` - Show process hierarchy/tree
- `pstree -p` - Process tree with PIDs

### Q29: What key columns should you understand in `ps aux` output?
**A:** Critical columns:
- **USER**: Process owner
- **%CPU**: CPU usage percentage
- **%MEM**: Memory usage percentage
- **VSZ**: Virtual memory size
- **RSS**: Physical memory usage
- **START**: Process start time
- **STAT**: Process status code

---

## ps Command Cheatsheet

```bash
# Basic usage
ps                      # Current terminal processes only
ps x                    # All user processes
ps aux                  # All processes with full details

# Sorting and filtering
ps aux --sort=-%cpu     # Sort by CPU usage (highest first)
ps aux --sort=-%mem     # Sort by memory usage (highest first)
ps aux | grep process   # Find specific process

# Process hierarchy
ps -ejH                 # Show hierarchy with indentation
ps axjf                 # Show process tree with lines

# Custom output
ps -o pid,ppid,user,comm    # Custom columns
ps -p PID                   # Specific process by PID
```

---

## Cgroup Slices and Resource Management

### Q30: What are cgroup slices in systemd?
**A:** Slices are hierarchical organizational units that group related services together in the cgroup tree. They create containers for processes to enable resource management and monitoring.

### Q31: What are the three main default slices?
**A:** 
- **`-.slice`** - Root slice containing everything
- **`system.slice`** - All systemd system services
- **`user.slice`** - All user sessions and processes

### Q32: Do slices reserve dedicated CPU/memory resources?
**A:** No! By default, all slices have unlimited resources (`infinity`). They are organizational containers, not resource reservations. All processes compete for the same total system resources.

### Q33: How do you determine which slice a process belongs to?
**A:** The slice is determined by **who started the process**:
- **System slice**: Processes started by systemd (PID 1) during boot
- **User slice**: Processes started by users in their sessions
- Check with `systemctl status service` or `systemd-cgls`

### Q34: Why are nginx and sshd in system.slice even though they run as different users?
**A:** Because they were **started by systemd** during boot, not by users. The slice is determined by the starter, not the final user ownership. Even if nginx drops privileges to run as user `nginx`, the service itself was started by systemd.

### Q35: How can you view the cgroup hierarchy and resource usage?
**A:** Key commands:
- `systemd-cgls` - Show complete cgroup tree
- `systemd-cgtop` - Real-time resource usage by cgroup
- `systemctl list-units --type=slice` - List all slices
- `systemctl show slice-name` - Show slice configuration

### Q36: When would you set resource limits on slices?
**A:** In production environments for:
- **Web servers**: Limit memory to prevent OOM
- **Multi-tenant systems**: Ensure fair resource sharing
- **Container orchestration**: Resource isolation
- **Critical services**: Guarantee minimum resources

### Q37: How do you create a custom slice with resource limits?
**A:** Create a slice unit file:
```bash
# /etc/systemd/system/myapp.slice
[Unit]
Description=My Application Slice

[Slice]
CPUQuota=50%
MemoryLimit=1G
TasksMax=100
```

### Q38: What's the relationship between slices and user sessions?
**A:** Each user login creates a `user-UID.slice` under `user.slice`:
- SSH sessions go into `session-N.scope`
- User systemd services go into `user@UID.service`
- All share the same system resources by default

---

## Process Priority and Niceness

### Q39: What's the relationship between Priority (PR) and Nice Value (NI)?
**A:** The formula is: **PR = 20 + NI** (in top display)
- **PR (Priority)**: Kernel's internal priority (lower number = higher priority)
- **NI (Nice Value)**: User-adjustable offset (-20 to +19)
- **Default**: NI = 0, so PR = 20

### Q40: Why do we need both priority and niceness concepts?
**A:** **Separation of concerns**:
- **Priority (PR)**: What the kernel scheduler actually uses internally
- **Niceness (NI)**: Human-friendly interface for adjusting process behavior
- **Design philosophy**: "Nice people finish last" - positive nice = lower priority

### Q41: How do you remember the niceness concept?
**A:** **Memory trick**: "Nice = Polite = Gives Way = Lower Priority"
- **High nice (+19)**: Very polite person, steps aside for others = **Low priority**
- **Low nice (-20)**: Rude person, pushes to front = **High priority**
- **Rule**: The nicer (more positive) the number, the more the process yields CPU

### Q42: What are real-world scenarios for using nice/renice?
**A:** Common use cases:
- **Database backups**: `nice +15 pg_dump` - Don't slow down web traffic
- **Video processing**: `nice +10 ffmpeg` - Keep system responsive during encoding
- **System emergency**: `renice -10 $(pgrep nginx)` - Boost web server priority
- **Code compilation**: `nice +5 make -j8` - Compile without freezing desktop

### Q43: When does niceness NOT help?
**A:** Niceness is ineffective for:
- **I/O bound processes** - Disk/network bottlenecks, not CPU competition
- **Low system load** - No CPU competition exists
- **Memory-intensive apps** - Nice doesn't control RAM usage
- **Real-time requirements** - Use `chrt` for real-time scheduling instead

### Q44: What happens during CPU resource contention?
**A:** **Without nice adjustment**: All processes compete equally, everything slows down
**With nice adjustment**: Critical processes get larger CPU time slices, background tasks get smaller slices
- **Total CPU stays 100%**, but allocation changes based on priority
- **High priority** processes become more responsive
- **Low priority** processes run slower but don't block critical functions

---

## Signals and Process Communication

### Q45: What is the core problem that signals solve?
**A:** Signals provide **inter-process communication** for process control:
- Stop runaway processes
- Gracefully shut down services  
- Reload configuration without restart
- Handle system events (logout, shutdown)
- Enable cooperative process management

### Q46: What are the main categories of signals?
**A:** **Termination Signals**:
- `SIGTERM` (15) - "Please exit cleanly" (save files, close connections)
- `SIGKILL` (9) - "Die immediately" (cannot be caught or ignored)
- `SIGQUIT` (3) - "Quit with core dump" (for debugging)

**Control Signals**:
- `SIGSTOP` (19) - Pause process (like Ctrl+Z)
- `SIGCONT` (18) - Resume paused process
- `SIGHUP` (1) - "Terminal disconnected" or "reload config"

**User Interaction**:
- `SIGINT` (2) - Interrupt (Ctrl+C)
- `SIGTSTP` (20) - Terminal stop (Ctrl+Z)

### Q47: Why is the signal design considered elegant?
**A:** **Asynchronous Communication**: 
- Signals don't block the sender
- Process handles signal when convenient
- **Process Choice**: Can handle, ignore (if allowed), or use default behavior
- **Cooperative**: Processes participate in their own lifecycle management

### Q48: What are practical examples of signal usage?
**A:** **Graceful Service Management**:
```bash
kill -HUP $(pgrep nginx)    # Reload config without downtime
kill -USR1 process_pid      # Trigger debug info dump
```

**System Shutdown Sequence**:
1. Send SIGTERM to all processes (please exit)
2. Wait 10 seconds
3. Send SIGKILL to remaining processes (force exit)

**Process Debugging**:
- `kill -USR1/USR2` for custom debugging actions
- Better than killing and restarting for investigation

### Q49: What's the difference between SIGTERM and SIGKILL?
**A:** 
- **SIGTERM (15)**: Polite request to terminate, process can clean up (save files, close connections, custom handlers)
- **SIGKILL (9)**: Immediate termination by kernel, cannot be caught, ignored, or handled
- **Best practice**: Always try SIGTERM first, use SIGKILL only as last resort

### Q50: How do signals enable zero-downtime service management?
**A:** Many services respond to specific signals for operational tasks:
- **SIGHUP**: Reload configuration files without stopping service
- **SIGUSR1/USR2**: Application-specific functions (log rotation, status dumps)
- **Avoids**: stop → edit config → start cycle that causes service interruption
- **Result**: Configuration changes applied instantly without dropping connections

---

## Practical Commands and Examples

### Q51: How do you create a CPU stress test with lower priority?
**A:** 
```bash
nice -n 5 dd if=/dev/zero of=/dev/null &
```
This creates infinite CPU load at lower priority (+5 niceness) to test system behavior without freezing the system.

### Q52: What's the complete workflow for emergency process priority adjustment?
**A:**
```bash
# Identify problem processes
top -o %CPU

# Boost critical service priority (requires root)
sudo renice -10 $(pgrep nginx)
sudo renice -10 $(pgrep postgres)

# Lower background task priority
sudo renice +15 $(pgrep backup)
sudo renice +19 $(pgrep logrotate)

# Monitor results
systemd-cgtop
```

---

## Zombie Processes and Process States

### Q53: What is a zombie process?
**A:** A **zombie** (defunct) is a process that has **finished executing** but still has an entry in the process table because its **parent hasn't read its exit status** yet.
- **Status**: Shows as 'Z' in ps output or `<defunct>` in command field
- **Memory usage**: Zero (0 RSS, 0 VSZ) - process is dead
- **Purpose**: Holds exit status until parent calls wait()

### Q54: How do you identify zombie processes?
**A:** Multiple detection methods:
```bash
ps aux | grep -i defunct                    # Look for <defunct>
ps aux | awk '$8 ~ /^Z/ { print $2 }'      # Status 'Z' processes
top                                         # Shows zombie count in summary
ps -eo pid,ppid,state,comm | grep " Z "    # State column 'Z'
```

### Q55: Why do zombie processes occur?
**A:** **Bad parent programming** - parent process doesn't clean up after child:
- **Child completes** execution and exits
- **Parent continues** without calling wait() to read exit status
- **Child becomes zombie** until parent reads its status
- **Common causes**: Scripts with background jobs, poorly written daemons

### Q56: How do you eliminate zombie processes?
**A:** **Method 1**: Kill the parent process (most effective)
```bash
ps -eo pid,ppid,state,comm | grep " Z "    # Find zombie's parent
kill -TERM parent_pid                       # Kill parent
# systemd adopts zombie → automatically cleans it up
```

**Method 2**: Signal parent to clean up
```bash
kill -CHLD parent_pid    # Send child signal to parent
```

**Method 3**: Restart the service
```bash
systemctl restart service_name
```

### Q57: Are zombie processes harmful?
**A:** **Usually NOT harmful** because:
- **No resource usage**: 0 memory, 0 CPU
- **Only consume**: One PID entry in process table
- **Limited impact**: Process table holds ~32,768 entries

**Becomes problematic** when thousands exist (PID exhaustion):
```bash
# Zombie bomb scenario
ps aux | grep defunct | wc -l
# 15000 zombies! - Cannot create new processes
```

---

## Process Termination Behavior Differences

### Q58: How does process termination behavior differ between older Linux and RHEL 9?
**A:** **Older Unix/Linux**: Killing parent → all children die with parent
**RHEL 9**: Killing parent → children become orphans (adopted by systemd)

**Design philosophy change**:
- **Old**: "Family death" - aggressive cascading termination
- **New**: "Individual death" - only targeted process dies, more explicit control

### Q59: Why does killing nginx master kill all workers in RHEL 9?
**A:** This is **application-specific coordination**, not system behavior:
- **nginx master** handles SIGTERM by actively signaling all workers to terminate
- **Coordinated shutdown**: Master waits for workers to exit, then exits itself
- **This is good design**: Ensures graceful service shutdown and no orphaned workers

**Process tree example**:
```bash
nginx: master (PID 1234) ← kill this
├── nginx: worker (1235) ← master tells it to exit  
├── nginx: worker (1236) ← master tells it to exit
└── nginx: worker (1237) ← master tells it to exit
```

---

## kill vs killall Command Comparison

### Q60: When should you use kill vs killall?
**A:** **Use `kill`** for:
- **Specific PID targeting**: `kill 1234`
- **Safety/precision**: Know exactly what you're killing
- **Different signals per process**: `kill -HUP 1234; kill -TERM 5678`

**Use `killall`** for:
- **Multiple instances**: `killall firefox` (all firefox processes)
- **Name-based killing**: Don't need to find PIDs first
- **Bulk cleanup**: `killall chrome` (cleanup all instances)

### Q61: What are the risks of using killall?
**A:** **Dangerous patterns**:
```bash
killall python     # Might kill system scripts!
killall bash       # Could kill your own shell!
killall java       # Might kill multiple applications
```

**Safer alternatives**:
```bash
pkill -f "specific_script.py"    # Pattern matching
pgrep firefox | xargs kill       # Two-step verification
systemctl stop service_name      # Proper service management
```

### Q62: Why is the "kill" command name misleading?
**A:** **Historical naming problem**: "kill" actually **sends signals**, not just kills:
```bash
kill -HUP nginx     # "Please reload config" (not killing!)
kill -STOP job      # "Pause yourself" (not killing!)
kill -USR1 apache   # "Rotate logs" (not killing!)
```

**Better names would be**: `signal`, `notify`, `message` - but too late to change due to backwards compatibility.

---

## USR1/USR2 Signal Applications

### Q63: What do USR1 and USR2 signals actually mean?
**A:** **USR1/USR2 = "User-defined signals"** - blank signals that applications can program for **any purpose**:
- **No standard meaning** across applications
- **Each application** documents what their USR1/USR2 do
- **Common misconception**: USR1 is "only for log rotation" (false!)

### Q64: How do different applications use USR1 differently?
**A:** **Web servers** (coincidentally similar):
```bash
kill -USR1 nginx        # Reopen log files
kill -USR1 apache       # Reopen log files
```

**Other applications** (completely different):
```bash
kill -USR1 postgresql   # Dump database statistics  
kill -USR1 redis        # Save database to disk
kill -USR1 rsyslog      # Flush log buffers
```

**Rule**: Always check application documentation - never assume USR1 meaning!

---

## References

- [Linux Crash Course - The ps Command](https://www.youtube.com/watch?v=wYwGNgsfN3I) - Comprehensive video tutorial covering ps command fundamentals, options, and practical usage