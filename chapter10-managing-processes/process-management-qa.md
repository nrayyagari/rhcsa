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