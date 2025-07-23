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
**A:** `pstree -p` shows the complete process tree with PIDs. You can also use `ps axo pid,ppid,tty,stat,comm` to see relationships.

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
pstree -p                 # Process tree
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

## References

- [Linux Crash Course - The ps Command](https://www.youtube.com/watch?v=wYwGNgsfN3I) - Comprehensive video tutorial covering ps command fundamentals, options, and practical usage