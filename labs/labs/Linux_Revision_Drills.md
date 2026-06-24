# Linux Revision Drills
### For DevOps Engineers — because everything is a Linux machine at the end of the day

---

## Module 1 — Navigation & File System

> The Linux filesystem is the foundation. You're always navigating it — inside containers, on EC2s, on K8s nodes.

---

**Drill 1: Navigate confidently**

```bash
pwd                  # where am i
ls -la               # list with hidden files + permissions
cd /etc              # absolute path
cd ../..             # relative path up two levels
cd ~                 # go home
cd -                 # go back to previous directory
```

---

**Drill 2: Understand the filesystem layout**

```
/               root of everything
/etc            config files (nginx.conf, systemd units, hosts)
/var/log        logs (nginx, syslog, app logs)
/tmp            temporary files, cleared on reboot
/home           user home directories
/opt            manually installed software
/proc           virtual FS — live kernel + process info
/sys            virtual FS — hardware/kernel tuning
/usr/bin        user binaries (most commands live here)
```

On a K8s node or EC2, you'll spend most time in `/etc`, `/var/log`, `/proc`, `/tmp`.

---

**Drill 3: Find files**

```bash
find / -name "nginx.conf"               # find by name
find /var/log -name "*.log" -mtime -1   # logs modified in last 1 day
find /app -type f -size +100M           # files over 100MB
locate nginx.conf                       # faster, uses index (run updatedb first)
which nginx                             # where is this binary?
type kubectl                            # is it a binary, alias, or function?
```

---

**Drill 4: File operations**

```bash
cp file.txt backup.txt
cp -r /app /app-backup          # recursive copy
mv old.txt new.txt              # move or rename
rm file.txt
rm -rf /tmp/testdir             # recursive force delete — be careful
ln -s /usr/bin/python3 python   # symlink (like PATH shortcuts)
```

---

## Module 2 — Text Processing

> 80% of DevOps work is reading logs, configs, and output. Master these and you'll debug 2x faster.

---

**Drill 5: Read files**

```bash
cat /etc/os-release             # print whole file
less /var/log/syslog            # paginate (q to quit, / to search)
head -20 /var/log/nginx/access.log    # first 20 lines
tail -50 /var/log/nginx/error.log     # last 50 lines
tail -f /var/log/nginx/error.log      # follow live (like docker logs -f)
```

---

**Drill 6: grep — your most used command**

```bash
grep "ERROR" app.log                        # find lines with ERROR
grep -i "error" app.log                     # case-insensitive
grep -r "DB_HOST" /etc/                     # recursive search in directory
grep -v "DEBUG" app.log                     # exclude lines with DEBUG
grep -n "timeout" app.log                   # show line numbers
grep -A 3 -B 3 "OOMKilled" app.log         # 3 lines after and before match
grep "ERROR\|WARN" app.log                  # multiple patterns
```

You'll use this constantly — searching logs on EC2s, inside containers, in config files.

---

**Drill 7: awk — extract columns**

```bash
awk '{print $1}' access.log               # print first column
awk '{print $1, $7}' access.log           # print columns 1 and 7
awk -F: '{print $1}' /etc/passwd          # use : as delimiter
awk '$9 == "500"' access.log              # filter where column 9 = 500
awk '{sum += $10} END {print sum}' access.log   # sum column 10
```

Great for parsing nginx access logs, CSV output, `kubectl get` output.

---

**Drill 8: sed — find and replace**

```bash
sed 's/localhost/0.0.0.0/g' nginx.conf          # replace in output
sed -i 's/localhost/0.0.0.0/g' nginx.conf       # replace in file (in-place)
sed -n '10,20p' app.log                         # print lines 10–20
sed '/^#/d' nginx.conf                          # delete comment lines
```

---

**Drill 9: Pipe and combine**

```bash
cat access.log | grep "POST" | awk '{print $7}' | sort | uniq -c | sort -rn | head -10
```

Top 10 most-hit POST endpoints. This is real-world log analysis in one line.

```bash
ps aux | grep nginx
kubectl get pods | grep -v Running
journalctl -u nginx | tail -100 | grep ERROR
```

---

## Module 3 — Processes & System Monitoring

> On a live EC2 or K8s node, you need to know what's eating CPU, memory, and disk.

---

**Drill 10: View running processes**

```bash
ps aux                          # all processes
ps aux | grep nginx             # filter for nginx
ps aux --sort=-%cpu | head -10  # top 10 by CPU
ps aux --sort=-%mem | head -10  # top 10 by memory
pgrep nginx                     # just get PIDs
```

---

**Drill 11: top and htop**

```bash
top                 # live process monitor
htop                # better UI (install if not present)
```

Inside `top`:
- `M` → sort by memory
- `P` → sort by CPU
- `k` → kill a process by PID
- `q` → quit

---

**Drill 12: Disk usage — critical for containers and nodes**

```bash
df -h                           # disk usage per filesystem
du -sh /var/log/*               # size of each item in /var/log
du -sh /* 2>/dev/null | sort -rh | head -10   # top 10 largest dirs
lsblk                           # list block devices
```

K8s nodes fill up with container logs and image layers. `df -h` is the first thing to check.

---

**Drill 13: Memory**

```bash
free -h                         # RAM and swap usage
cat /proc/meminfo               # detailed memory info
```

---

**Drill 14: Kill processes**

```bash
kill <PID>              # SIGTERM — graceful
kill -9 <PID>           # SIGKILL — force
killall nginx           # kill all processes named nginx
pkill -f "node index"   # kill by pattern match
```

---

**Drill 15: Background jobs**

```bash
command &               # run in background
jobs                    # list background jobs
fg %1                   # bring job 1 to foreground
bg %1                   # send to background
nohup command &         # keep running after logout
Ctrl+C                  # kill foreground process
Ctrl+Z                  # suspend foreground process
```

---

## Module 4 — Networking

> Debugging connectivity between services, pods, and AWS resources is a daily DevOps task.

---

**Drill 16: Check connectivity**

```bash
ping google.com                         # basic reachability
curl -I https://google.com              # HTTP headers
curl -v http://localhost:3000/health    # verbose — see full request/response
wget -O- http://localhost:3000          # alternative to curl
```

---

**Drill 17: DNS resolution**

```bash
nslookup google.com
dig google.com
dig +short google.com
cat /etc/resolv.conf            # what DNS servers is this machine using?
cat /etc/hosts                  # local hostname overrides
```

On K8s, `dig` inside a pod tells you if CoreDNS is resolving service names correctly.

---

**Drill 18: Open ports and sockets**

```bash
ss -tlnp                        # listening TCP ports (modern)
netstat -tlnp                   # same, older command
lsof -i :3000                   # what process is using port 3000?
lsof -i -P -n | grep LISTEN     # all listening ports with process names
```

---

**Drill 19: Network interfaces**

```bash
ip addr                         # all network interfaces and IPs
ip route                        # routing table
ip link                         # interface status (up/down)
```

On EC2 and K8s nodes, `ip addr` tells you the node's private IP and any CNI-assigned interfaces.

---

**Drill 20: Test port reachability**

```bash
telnet <host> <port>                        # basic TCP check
nc -zv <host> <port>                        # netcat — cleaner
curl -s telnet://<host>:<port>              # curl alternative
```

If a pod can't reach the DB, `nc -zv db-service 5432` tells you instantly if it's a network or app issue.

---

## Module 5 — Users, Permissions & Security

> Every container, service, and SSH connection involves users and permissions.

---

**Drill 21: File permissions**

```bash
ls -la /app

# Output:
# -rwxr-xr-- 1 appuser appgroup 1234 Jun 1 index.js
#  ^ ^ ^ ^
#  | | | └── other: r-- (read only)
#  | | └──── group: r-x (read + execute)
#  | └────── owner: rwx (read + write + execute)
#  └──────── file type: - (file), d (dir), l (symlink)

chmod 755 script.sh             # rwxr-xr-x
chmod +x script.sh              # add execute for everyone
chmod 600 ~/.ssh/id_rsa         # owner read/write only (required for SSH keys)
chown appuser:appgroup file.txt # change owner and group
```

---

**Drill 22: Users and groups**

```bash
whoami                          # current user
id                              # uid, gid, groups
cat /etc/passwd                 # all users
cat /etc/group                  # all groups
sudo su - appuser               # switch to appuser
useradd -m newuser              # create user
usermod -aG docker ubuntu       # add ubuntu user to docker group
```

In your Dockerfiles, you create non-root users — this is the underlying Linux mechanism.

---

**Drill 23: SSH**

```bash
ssh -i ~/.ssh/key.pem ubuntu@<ec2-ip>       # connect to EC2
ssh -i key.pem -L 8080:localhost:80 ubuntu@<ec2-ip>   # port forward
scp -i key.pem file.txt ubuntu@<ec2-ip>:/home/ubuntu/ # copy file to EC2
cat ~/.ssh/authorized_keys                  # who can SSH into this machine
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys   # required permissions
```

---

**Drill 24: sudo and privilege**

```bash
sudo command                    # run as root
sudo -i                         # open root shell
sudo -u appuser command         # run as specific user
visudo                          # safely edit sudoers file
cat /etc/sudoers.d/*            # custom sudo rules
```

---

## Module 6 — Systemd & Services

> EC2 instances and K8s nodes run services managed by systemd. Nginx, Docker daemon, kubelet — all systemd units.

---

**Drill 25: Manage services**

```bash
systemctl status nginx              # is it running?
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx              # reload config without downtime
systemctl enable nginx              # start on boot
systemctl disable nginx             # don't start on boot
systemctl list-units --type=service # list all services
```

---

**Drill 26: journalctl — systemd logs**

```bash
journalctl -u nginx                         # all logs for nginx
journalctl -u nginx -f                      # follow live
journalctl -u nginx --since "1 hour ago"    # last hour
journalctl -u kubelet -n 100                # last 100 lines
journalctl -p err -u docker                 # only errors
journalctl --disk-usage                     # how much disk logs are using
```

On EC2, `journalctl -u kubelet` is where you go when a node is NotReady.

---

**Drill 27: Write a systemd unit file**

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Node.js App
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production PORT=3000

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload             # reload after creating/editing unit
systemctl enable --now myapp        # enable + start immediately
```


---

## Module 7 — Shell Scripting

> Automation is your job. Shell scripts are the glue between everything.

---

**Drill 28: Script basics**

```bash
#!/bin/bash
set -euo pipefail       # exit on error, undefined var, pipe failure
                        # always put this at the top of scripts

NAME="Srinivas"
echo "Hello, $NAME"

# Conditionals
if [ -f "/etc/nginx/nginx.conf" ]; then
  echo "nginx config exists"
else
  echo "not found"
fi

# Loops
for pod in $(kubectl get pods -o name); do
  echo "Checking $pod"
done
```

---

**Drill 29: Useful script patterns**

```bash
# Check if command exists
if ! command -v docker &>/dev/null; then
  echo "docker not installed"
  exit 1
fi

# Read a file line by line
while IFS= read -r line; do
  echo "Processing: $line"
done < servers.txt

# Function
check_health() {
  local url=$1
  local status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  if [ "$status" == "200" ]; then
    echo "$url is healthy"
  else
    echo "$url returned $status"
  fi
}

check_health "http://localhost:3000/health"
```

---

**Drill 30: Environment and variables**

```bash
export MY_VAR="value"           # set env var for child processes
echo $MY_VAR
env                             # all env vars
printenv PATH                   # specific var
unset MY_VAR                    # remove it

# Default value if not set
PORT=${PORT:-3000}

# Read from file
export $(cat .env | xargs)      # load .env file into environment
```

---

## Module 8 — Performance & Troubleshooting

> When something is broken on a node, container, or EC2 — this is your toolkit.

---

**Drill 31: CPU and load**

```bash
uptime                          # load average (1, 5, 15 min)
nproc                           # number of CPU cores
lscpu                           # CPU details
vmstat 1 5                      # CPU, memory, IO every 1 sec, 5 times
iostat -x 1                     # disk IO stats
```

Load average > number of cores = system is overloaded.

---

**Drill 32: strace — what is this process doing?**

```bash
strace -p <PID>                 # trace system calls of running process
strace -e open,read,write ls    # trace specific syscalls
```

When a process hangs and you don't know why, `strace` shows you exactly what it's waiting on.

---

**Drill 33: lsof — open files and connections**

```bash
lsof -p <PID>                   # all files opened by process
lsof /var/log/app.log           # who has this file open?
lsof -i TCP:3000                # who is using port 3000?
lsof -u appuser                 # all files opened by appuser
```

---

**Drill 34: Troubleshooting checklist**

When something is broken on a Linux machine, check in this order:

```bash
# 1. Is the service running?
systemctl status myapp

# 2. What do the logs say?
journalctl -u myapp -n 100

# 3. Is the port listening?
ss -tlnp | grep 3000

# 4. Is disk full?
df -h

# 5. Is memory exhausted?
free -h

# 6. Is CPU maxed?
uptime && top

# 7. Can it reach the network?
curl -v http://dependency-service/health

# 8. DNS working?
dig dependency-service

# 9. Any kernel/OOM messages?
dmesg | tail -50 | grep -i "oom\|error\|fail"
```

---

## Quick Reference Cheatsheet

| Command | What it does |
|---|---|
| `tail -f /var/log/app.log` | Follow live logs |
| `grep -r "ERROR" /var/log/` | Search all logs for ERROR |
| `df -h` | Disk usage |
| `free -h` | Memory usage |
| `ss -tlnp` | Listening ports |
| `ps aux --sort=-%cpu` | Processes by CPU |
| `systemctl status nginx` | Service status |
| `journalctl -u nginx -f` | Follow service logs |
| `kill -9 <PID>` | Force kill process |
| `chmod 755 script.sh` | Set permissions |
| `ssh -i key.pem user@host` | SSH into server |
| `find / -name "*.conf"` | Find config files |
| `curl -v http://localhost/health` | Test HTTP endpoint |
| `dmesg | tail -50` | Kernel messages |
| `strace -p <PID>` | Trace process syscalls |

---

## Your Linux Mental Map

```
Everything is a file
    ↓
Processes read/write files
    ↓
The kernel manages processes, memory, and hardware
    ↓
systemd manages services (nginx, docker, kubelet)
    ↓
Your containers are just Linux processes with namespaces + cgroups
    ↓
Your K8s nodes are just Linux machines running kubelet
    ↓
Your EC2s are just Linux machines in AWS VPC
```

Linux is not a separate topic from Docker or K8s. It **is** Docker and K8s — one layer down.

---

*Master Linux and everything else becomes transparent. 🐧*
