```bash
ps -u "$(id -un)" -o pid,ppid,stat,etime,%cpu,%mem,args --sort=-%cpu
```