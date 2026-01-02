## 1. `chmod` – Change File Permissions

**Explanation:** Modifies file or directory permissions (read, write, execute).

**Options:**

- `u` → user (owner), `g` → group, `o` → others, `a` → all
    
- `+` → add, `-` → remove, `=` → assign exact
    
- `-R` → recursive
    

**Examples:**

```bash
chmod u+x script.sh         # give execute permission to owner
chmod 644 file.txt          # rw-r--r--
chmod -R 755 /var/www       # recursively set rwxr-xr-x
```

**When to use:**  
To set execution rights on scripts, adjust web server directories, or secure sensitive files.

---

## 2. `chown` – Change File Ownership

**Explanation:** Sets or changes the owner and group of files.

**Options:**

- `user:group` → specify both owner and group
    
- `-R` → recursive
    

**Examples:**

```bash
chown devops:devops file.txt
chown -R www-data:www-data /var/www/html
```

**When to use:**  
After deployments, to ensure the right user/service can access or execute files.

---

## 3. `ls` – List Files and Directories

**Explanation:** Displays directory contents.

**Options:**

- `-l` → long listing (details)
    
- `-a` → show hidden files
    
- `-h` → human-readable sizes
    
- `-t` → sort by modification time

**Examples:**

```bash
ls -lah
ls -ltr
```

**When to use:**  
To inspect files, check sizes, and verify recent changes.

---

## 4. `df` – Disk Usage (Filesystem)

**Explanation:** Shows disk space usage of mounted filesystems.

**Options:**

- `-h` → human-readable
    
- `-T` → show filesystem type
    

**Examples:**

```bash
df -h
df -Th
```

**When to use:**  
To monitor disk space before/after deployments, troubleshooting full disks.

---

## 5. `du` – Directory/File Size

**Explanation:** Shows disk usage for files and directories.

**Options:**

- `-h` → human-readable
    
- `-s` → summary only
    
- `--max-depth=N` → limit recursion depth
    

**Examples:**

```bash
du -sh /var/log/*
du -h --max-depth=1 .
```

**When to use:**  
To locate large files/folders when a partition is full.

---

## 6. `systemctl` – Manage Systemd Services

**Explanation:** Controls services and processes in systemd.

**Options:**

- `start` / `stop` / `restart`
    
- `status` → check service state
    
- `enable` / `disable` → start on boot
    

**Examples:**

```bash
systemctl status nginx
systemctl restart docker
systemctl enable sshd
```

**When to use:**  
Managing application services, restarting after config changes, enabling daemons.

---

## 7. `journalctl` – View System Logs

**Explanation:** Queries and displays logs collected by systemd.

**Options:**

- `-u <service>` → filter by service
    
- `-f` → follow logs (like tail -f)
    
- `--since "10 min ago"` → time filter
    

**Examples:**

```bash
journalctl -u nginx -f
journalctl --since "2025-08-31 09:00"
```

**When to use:**  
Troubleshooting service issues and checking logs after restarts.

---

## 8. `tar` – Archive Files

**Explanation:** Create and extract compressed archives.

**Options:**

- `-c` → create, `-x` → extract
    
- `-v` → verbose, `-f` → filename
    
- `-z` → gzip, `-j` → bzip2
    

**Examples:**

```bash
tar -czvf backup.tar.gz /var/www
tar -xzvf backup.tar.gz -C /tmp
```

**When to use:**  
For backups, packaging artifacts, or transferring files.

---

## 9. `scp` – Secure Copy

**Explanation:** Copies files between servers over SSH.

**Options:**

- `-r` → recursive copy
    
- `-P` → specify port
    

**Examples:**

```bash
scp file.txt user@server:/tmp/
scp -r /var/www user@server:/backup/
```

**When to use:**  
Moving files, configs, or backups across servers securely.

---

## 10. `rsync` – Sync Files/Directories

**Explanation:** Efficiently copies/syncs files, only transferring differences.

**Options:**

- `-a` → archive mode (preserves perms, symlinks)
    
- `-v` → verbose
    
- `--delete` → remove extra files at destination
    

**Examples:**

```bash
rsync -av /var/www/ user@server:/backup/www/
rsync -avz --delete ./build/ user@server:/var/www/
```

**When to use:**  
Deployments, syncing configs, and backups.

---

## 11. `grep` – Search Inside Files

**Explanation:** Searches text using patterns.

**Options:**

- `-i` → ignore case
    
- `-r` → recursive
    
- `-n` → show line numbers
    

**Examples:**

```bash
grep "error" /var/log/syslog
grep -rn "password" /etc/
```

**When to use:**  
Log analysis, searching configs, debugging.

---

## 12. `ps` & `top` – Process Management

**Explanation:** View running processes and system resource usage.

**Options (ps):**

- `aux` → show all processes
    
- `-ef` → full details
    

**Examples:**

```bash
ps aux | grep nginx
top
htop   # (if installed)
```

**When to use:**  
Finding process IDs, checking memory/CPU usage, monitoring system health.

---

## 13. `kill` – Terminate Processes

**Explanation:** Sends signals to processes (default is `TERM`).

**Options:**

- `-9` → force kill
    
- `-15` → graceful termination
    

**Examples:**

```bash
kill -9 1234
pkill -f "python script.py"
```

**When to use:**  
Stopping stuck services or runaway processes.

---

## 14. `netstat` / `ss` – Network Connections

**Explanation:** Shows network sockets and ports.

**Options (`ss` recommended):**

- `-tuln` → listening ports
    
- `-p` → show process
    

**Examples:**

```bash
ss -tuln
ss -plnt | grep 8080
```

**When to use:**  
Check if a service is listening, troubleshoot port conflicts.
