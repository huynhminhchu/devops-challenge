# 🚀 Troubleshooting High Memory Usage on Ubuntu 24.04 VM Running NGINX Load Balancer

# Troubleshoot steps

## 1. Verify the issue:
- Log into the VM via SSH.
- Check current memory usage to confirm `99%` usage:
    ```bash
    htop
    ```

    ```bash
    ps aux --sort=-%mem | head -n 1
    ```

- Confirm `nginx` is running:
    ```bash
    sudo systemctl status nginx
    ```
- Check NGINX’s memory footprint:
    ```bash
    ps -C nginx -o %mem,rss,vsz
    ```

## 2. Analyze system logs:

- Check system logs (`/var/log/messages` or `/var/log/syslog`) and `nginx` log
    ```bash
    tail -f /var/log/messages
    tail -f /var/log/nginx/error.log
    tail -f /var/log/nginx/access.log
    dmesg | grep -i "oom"
    journalctl -xe
    ```

## 3. Examine resource trends:

- If possible, take advantage monitoring tools like of `Grafana`, `Prometheus` to observe the memory usage patterns over time.
- Check for memory leaks or spikes correlating with traffic.

## 4. Gather system contexts:

- Check disk usage:
    ```bash
    df -h
    ```

- Verify swap usage:
    ```bash
    swapon --show
    ```

## 5. Review configuration

- Inspect `nginx` configuration (`/etc/nginx/nginx.conf` and site configs in `/etc/nginx/sites-enabled/`) for misconfigurations.

    ```bash
    vim /etc/nginx/nginx.conf
    ```

# Scenario Analysis
📌 Since this VM is only running a NGINX load balancer, it appears that there can be several plausible issues causing the 99% memory usage:

🛑 NGINX worker processes misconfiguration
🛑 Memory leak in nginx or Upstream Proxying
🛑 Memory fragmentation in Linux


## 🌩️ Issue 1: Nginx Worker Processes Misconfiguration

### 🔎 Possible Causes
✅ NGINX’s **worker_processes** is set to a high number (ex: `auto` or a manually large value like `32`), and each worker is handling many connections, leading to excessive memory allocation.

✅ NGINX's **worker_connections** directive is also set too high (ex: 10000), causing NGINX to reserve memory for a massive number of potential connections, even if traffic is moderate.

### ⚠️ Impact

- Upstream services lag as NGINX chokes.
- Risk of crashes or OOM intervention, VM freezes.

### 🛠️ Recovery Steps

- **Tune Worker Processes**: Edit `/etc/nginx/nginx.conf` and set `worker_processes` to match CPU cores. Fewer workers = less memory overhead.

- **Adjust Worker Connections**: Check `worker_connections` (default is 1024). Calculate max connections (`worker_processes` × `worker_connections`) and scale it down:
    - If traffic averages 500 concurrent connections, set `worker_connections` to ~150 per worker (e.g., 600 total for 4 workers).
    - Use `ss -tuln | wc -l` to estimate active connections and adjust conservatively.
- Verify `nginx` configuration and reload:
    ```bash
    sudo nginx -t
    sudo nginx -s reload
    ```
- Monitor with `free -h` to make sure memory usage should drop as fewer connections are pre-allocated.
## 🌩️ Issue 2: Memory Leak or Buffering Overload

### 🔎 Possible Causes
✅ NGINX has a memory leak (bug in version/module) or buffers large upstream responses in memory due to oversized **proxy_buffer_size/proxy_buffers**.
Proxying 10MB responses with default buffers fills RAM over time.

### ⚠️ Impact

- Upstream services lag as NGINX chokes.
- Risk of crashes or OOM intervention, VM freezes
- Delayed request handling.

### 🛠️ Recovery Steps

- **Reduce buffer sizes**: Edit `/etc/nginx/nginx.conf` and set `proxy_buffer_size` to a smaller numbers: `proxy_buffer_size 4k` or `proxy_buffer_size 8k`.

- Verify `nginx` configuration and reload:
    ```bash
    sudo nginx -t
    sudo nginx -s reload
    ```

## 🌩️ Issue 3: Memory Fragmentation

### 🔎 Possible Causes
✅ The VM might be experiencing memory fragmentation, causing inefficient memory allocation.

### ⚠️ Impact

- Upstream services lag as NGINX chokes.
- Risk of crashes or OOM intervention, VM freezes
- Delayed request handling.

### 🛠️ Recovery Steps

- **Clear memory cache**:

```bash
sync; echo 3 > /proc/sys/vm/drop_caches
```