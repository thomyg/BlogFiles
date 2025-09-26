🧙🏾‍♂️: **Production-Ready Runbook — Intermittent 502s on NGINX → Node.js (Linux)**  
_No questions. Proceeding autonomously with a complete, actionable plan._

---

## 1) 🔀 Decision Tree (crisp)

> **Run this once to collect recent 502 context:**
```bash
# Collect 502s + upstream errors from last 2 hours
sudo bash -c '
echo "===== NGINX 502 ACCESS HITS (2h) =====";
journalctl -u nginx --since "2 hours ago" --no-pager 2>/dev/null | cat >/dev/null || true;
awk -v d="$(date -d "2 hours ago" "+%d/%b/%Y:%H:%M")" \
    "$4>=\"[\"d && $9==502" /var/log/nginx/access.log 2>/dev/null | tail -n 200;
echo; echo "===== NGINX ERROR LOG (relevant) (2h) =====";
grep -Ei "upstream|502|connect\(\)|timed out|too big header|no live upstreams|prematurely closed|SSL_do_handshake|cannot assign requested address" \
    /var/log/nginx/error.log 2>/dev/null | tail -n 200;
echo; echo "===== NGINX ACTIVE CONFIG (proxy-related) =====";
sudo nginx -T 2>/dev/null | grep -E "proxy_(connect|read|send|buffers|buffer_size|busy_buffers|http_version|set_header|ssl_server_name)|upstream|keepalive" -n
'
```

**A.** If error log shows `connect() failed (111: Connection refused)` or `no live upstreams` → **Branch A: Upstream down/port wrong**

**B.** If error log shows `upstream timed out (110)` while **connecting** or **reading response header** → **Branch B1: timeout connecting** (network/listener backlog) or **Branch B2: timeout reading header** (slow Node/DB/GC)

**C.** If error log shows `upstream prematurely closed connection` → **Branch C: app closed connections / keepalive mismatch / crash**

**D.** If error log shows `upstream sent too big header` → **Branch D: headers/cookies exceed NGINX buffers**

**E.** If error log shows `SSL_do_handshake() failed` or `upstream SSL` errors → **Branch E: TLS SNI/verify mismatch**

**F.** If error log shows `connect() failed (99: Cannot assign requested address)` → **Branch F: ephemeral ports exhaustion**

**G.** None of the above clear? → **Branch G: capacity/FD limits**, check connection & FD ceilings

---

## 2) 📟 Exact Commands & Log Locations per Branch

### Branch A — Upstream down / wrong port / process flapping
```bash
# Find Node upstreams from NGINX config (IPs:ports)
sudo nginx -T 2>/dev/null | awk '
  /^[ \t]*upstream[ \t]+/ {u=$2}
  /^[ \t]*server[ \t]+([0-9]+\.){3}[0-9]+:[0-9]+/ {gsub(";",""); if(u!=""){print $2}}
' | sort -u

# Probe every discovered upstream (replace with output if any), otherwise try localhost:3000
for u in $(sudo nginx -T 2>/dev/null | awk '\''/server [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+/{gsub(";","");print $2}'\'' | sort -u); do
  echo "Probing http://$u/health or /:";
  curl -sS -m 2 "http://$u/health" -o /dev/null -w "HTTP:%{http_code} time:%{time_total}s\n" || true
  curl -sS -m 2 "http://$u/"       -o /dev/null -w "HTTP:%{http_code} time:%{time_total}s\n" || true
done
curl -sS -m 2 "http://127.0.0.1:3000/" -o /dev/null -w "HTTP:%{http_code} time:%{time_total}s\n" || true

# Is Node running? (systemd + PM2)
systemctl --type=service --no-pager | grep -Ei 'node|pm2|app|api' || true
command -v pm2 >/dev/null && pm2 ls || true
journalctl -u nginx -n 100 --no-pager
journalctl -u pm2 -n 200 --no-pager 2>/dev/null || true
journalctl -u node -n 200 --no-pager 2>/dev/null || true
```

### Branch B1 — Timeout **connecting** to upstream (SYN backlog, worker saturation, network)
```bash
# Network backlog & SYNs
ss -s
ss -Htan state syn-recv | wc -l
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog

# NGINX worker connections and OS nofile
sudo nginx -T 2>/dev/null | grep -E "worker_processes|worker_connections|multi_accept"
ulimit -n
cat /proc/sys/fs/file-nr

# Node listening sockets & backlog
sudo ss -ltnp | grep -E "node|3000|:3[0-9]{3}" || true
```

### Branch B2 — Timeout **reading response header** (slow Node/DB/GC/blocking)
```bash
# Do a few slow probes end-to-end and to upstream directly
time curl -sS -m 10 -H 'Connection: close' -o /dev/null -w "HTTP:%{http_code} time:%{time_total}s\n" http://localhost/
time curl -sS -m 10 -H 'Connection: close' -o /dev/null -w "HTTP:%{http_code} time:%{time_total}s\n" http://127.0.0.1:3000/

# Node event loop lag (if PM2)
command -v pm2 >/dev/null && pm2 monit || true
command -v pm2 >/dev/null && pm2 logs --lines 200 || true

# CPU, load, GC pauses signal
uptime
top -b -n1 | head -n 20
journalctl -u node --since "30 min ago" --no-pager 2>/dev/null | tail -n 200 || true
```

### Branch C — Upstream **prematurely closed** connection
```bash
# Look for crashes/restarts and Keep-Alive mismatches
grep -i "prematurely closed" /var/log/nginx/error.log | tail -n 50
journalctl -u node --since "2 hours ago" --no-pager 2>/dev/null | tail -n 200 || true
command -v pm2 >/dev/null && pm2 logs --lines 200 || true

# Check NGINX upstream HTTP/1.1 keepalive setup
sudo nginx -T 2>/dev/null | grep -E "proxy_http_version|proxy_set_header Connection|keepalive"
```

### Branch D — **Too big header** from upstream
```bash
grep -i "too big header" /var/log/nginx/error.log | tail -n 50
# Sample big Set-Cookie in access log
grep -F " 502 " /var/log/nginx/access.log | tail -n 50
```

### Branch E — **TLS** to upstream (SNI/verify)
```bash
grep -Ei "SSL_do_handshake|upstream SSL|certificate|verify" /var/log/nginx/error.log | tail -n 100
sudo nginx -T 2>/dev/null | grep -E "proxy_pass https|proxy_ssl_server_name|proxy_ssl_verify"
```

### Branch F — **Ephemeral ports exhaustion** (NGINX-as-client)
```bash
# Look for "Cannot assign requested address" in errors
grep -i "Cannot assign requested address" /var/log/nginx/error.log | tail -n 50

# Port range, TIME_WAIT pressure
sysctl net.ipv4.ip_local_port_range net.ipv4.tcp_fin_timeout
ss -Htan state time-wait | wc -l
```

### Branch G — **Capacity / FD limits / per-worker caps**
```bash
# NGINX current limits
sudo nginx -T 2>/dev/null | grep -E "worker_processes|worker_connections"
ulimit -n
cat /proc/sys/fs/file-max

# Node FD usage
pgrep -f node | xargs -r -I{} sh -c 'echo "PID {}: $(ls /proc/{}/fd | wc -l) FDs"'
```

**Log locations** (default):
- NGINX: `/var/log/nginx/access.log`, `/var/log/nginx/error.log`, `journalctl -u nginx`
- Node (systemd): `journalctl -u <service>`
- Node (PM2): `pm2 logs`, `~/.pm2/logs/*`

---

## 3) 🧩 Probable Root Causes (ranked, with % confidence)

> Based on typical production patterns for NGINX→Node:

1) **Slow Node (DB latency, event-loop blocking, GC pauses)** → `upstream timed out while reading response header` **(35%)**  
2) **Node process down/restarting / wrong port** → `connect() failed (111: Connection refused)` / `no live upstreams` **(20%)**  
3) **Large headers/cookies exceed proxy buffers** → `upstream sent too big header` **(15%)**  
4) **NGINX/OS FD or connection ceilings** → many 502s under load; `worker_connections`/`ulimit` low **(10%)**  
5) **Keep-Alive mismatch / premature close** → `upstream prematurely closed connection` **(8%)**  
6) **Upstream TLS SNI/verify mismatch** → `SSL_do_handshake() failed` **(4%)**  
7) **Ephemeral port exhaustion (Cannot assign requested address)** **(5%)**  
8) **DNS/upstream name resolution quirks** (names change; no `resolve`) **(3%)**

Total: 100%

---

## 4) 🛡️ One Safe One‑Liner Mitigation per Cause (apply + reload)

> These are **additive** drop-in configs or safe kernel tunings, designed to be **reversible** and **low-risk**. Each is followed by a syntax test and reload where applicable.

### 4.1 Slow Node / timeouts while **reading response header** (Cause #1)
```bash
# Increase upstream timeouts, enable keepalive, and prevent 'Connection: close' to upstream (safe defaults)
sudo bash -c 'cat >/etc/nginx/conf.d/zz_timeouts_keepalive.conf <<EOF
proxy_connect_timeout 5s;
proxy_read_timeout 120s;
proxy_send_timeout 120s;
proxy_http_version 1.1;
proxy_set_header Connection "";
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.2 Upstream down / connection refused / flapping (Cause #2)
```bash
# Ensure Node stays up: PM2 or systemd (run both safely; whichever exists will apply)
command -v pm2 >/dev/null && (pm2 resurrect 2>/dev/null || pm2 save; pm2 startup -u "$USER" --hp "$HOME" >/dev/null || true)
sudo bash -c 'printf "[Service]\nRestart=always\nRestartSec=2\n" > /etc/systemd/system/node.service.d/override.conf' 2>/dev/null || true
sudo systemctl daemon-reload 2>/dev/null || true
# NGINX: fail fast to down instances to reduce 502 bursts
sudo bash -c 'cat >/etc/nginx/conf.d/zz_upstream_failfast.conf <<EOF
proxy_next_upstream error timeout http_502 http_504;
proxy_next_upstream_tries 2;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.3 Large headers/cookies (Cause #3)
```bash
# Increase proxy buffers to accommodate large upstream headers safely
sudo bash -c 'cat >/etc/nginx/conf.d/zz_proxy_buffers.conf <<EOF
proxy_buffer_size 32k;
proxy_buffers 8 64k;
proxy_busy_buffers_size 128k;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.4 FD / connection ceilings (Cause #4)
```bash
# Raise NGINX file descriptors and worker connections conservatively
sudo mkdir -p /etc/systemd/system/nginx.service.d
sudo bash -c 'printf "[Service]\nLimitNOFILE=131072\n" > /etc/systemd/system/nginx.service.d/override.conf'
sudo systemctl daemon-reload && sudo systemctl restart nginx
# If worker_connections is small, bump via drop-in (http-level include)
sudo bash -c 'cat >/etc/nginx/conf.d/zz_worker_connections.conf <<EOF
# Effective ceiling ~= worker_processes * worker_connections
events { worker_connections 8192; }
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.5 Keep-Alive mismatch / premature close (Cause #5)
```bash
# Ensure upstream keepalive is enabled (if using 'upstream' blocks) and headers are correct
sudo bash -c 'cat >/etc/nginx/conf.d/zz_upstream_keepalive.conf <<EOF
proxy_http_version 1.1;
proxy_set_header Connection "";
# If upstream uses 'upstream NAME { ... }', this enables shared keepalive pools:
# keepalive 64;  # (only valid inside an upstream{} block; harmless here if unused)
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.6 Upstream TLS SNI/verify mismatch (Cause #6)
```bash
# Pass SNI to upstream; do not disable verification unless you control trust chain
sudo bash -c 'cat >/etc/nginx/conf.d/zz_upstream_tls.conf <<EOF
proxy_ssl_server_name on;
# To trust internal CAs, uncomment and supply your bundle:
# proxy_ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;
# proxy_ssl_verify on;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

### 4.7 Ephemeral port exhaustion (Cause #7)
```bash
# Safely widen client port range and reduce FIN timeout
sudo sysctl -w net.ipv4.ip_local_port_range="10240 65535" net.ipv4.tcp_fin_timeout=15
```

### 4.8 DNS re-resolution for named upstreams (Cause #8)
```bash
# Provide resolvers with caching; effective immediately for configs using 'resolve' or variables
sudo bash -c 'cat >/etc/nginx/conf.d/zz_resolver.conf <<EOF
resolver 1.1.1.1 9.9.9.9 valid=300s;
resolver_timeout 5s;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

> **Revert any drop-in:** remove the created file in `/etc/nginx/conf.d/` and reload.

---

## 5) 🛠️ Fast “Stop-the-Bleed” Bundle (optional but useful)

```bash
# Apply conservative, broadly safe mitigations at once; then reload
sudo bash -c 'cat >/etc/nginx/conf.d/zz_502_safe_defaults.conf <<EOF
proxy_connect_timeout 5s;
proxy_read_timeout 120s;
proxy_send_timeout 120s;

proxy_http_version 1.1;
proxy_set_header Connection "";

proxy_buffer_size 32k;
proxy_buffers 8 64k;
proxy_busy_buffers_size 128k;

proxy_next_upstream error timeout http_502 http_504;
proxy_next_upstream_tries 2;

resolver 1.1.1.1 9.9.9.9 valid=300s;
resolver_timeout 5s;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

---

## 6) 🧭 Verification (prove 502s are gone)

```bash
# Warm-up and smoke-test through NGINX
for i in {1..10}; do curl -sS -o /dev/null -w "HTTP:%{http_code} t=%{time_total}s\n" http://localhost/; done

# Concurrency test (light)
seq 1 50 | xargs -n1 -P10 -I{} curl -sS -o /dev/null -w "%{http_code}\n" http://localhost/ | sort | uniq -c

# Re-check for fresh 502s
grep -F " 502 " /var/log/nginx/access.log | tail -n 50 || true
```

---

## 7) 🧾 “Root-Cause by Signature” Quick Map

- `connect() failed (111: Connection refused)` / `no live upstreams` → **Node down/port** → Apply **4.2**, check Node service/PM2.
- `upstream timed out while connecting` → **backlog/FDs** → **4.4**; see **B1**.
- `upstream timed out while reading response header` → **slow app** → **4.1**; see **B2**, inspect Node/DB.
- `upstream prematurely closed connection` → **crash/keepalive** → **4.5**, check logs.
- `upstream sent too big header` → **buffers/cookies** → **4.3**, reduce cookie bloat if possible.
- `SSL_do_handshake() failed` → **SNI/verify** → **4.6**.
- `Cannot assign requested address` → **ephemeral ports** → **4.7**.

---

## 8) 📚 Minimal Ongoing Observability (non-intrusive)

```bash
# Add request IDs for correlation (access+error logs)
sudo bash -c 'cat >/etc/nginx/conf.d/zz_logging.conf <<EOF
map \$request_id \$reqid { default \$request_id; }
log_format main_ext '\''$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent '
                     '\''"$http_referer" "$http_user_agent" reqid=$reqid upstream=$upstream_addr '
                     '\''rt=$request_time urt=$upstream_response_time'\'' ;
access_log /var/log/nginx/access.log main_ext;
EOF'
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9) ✅ What “Done” Looks Like

- No new 502s in access/error logs across peak hour.  
- Health probes return 200 with stable latency (p95 within SLO).  
- Node service stable (no restarts/crashes), CPU/FD within budget.  
- NGINX `nginx -T` shows the applied mitigations; config passes `nginx -t`.

---

## 10) 🧠 Assumptions Made

- Linux distro uses **systemd** (NGINX unit name `nginx`).
- NGINX includes `/etc/nginx/conf.d/*.conf` inside `http{}` (common on Debian/Ubuntu/CentOS/Alma/RHEL).
- Node upstream is HTTP on `localhost:3000` **or** declared in `upstream{}`; health endpoint is `/health` (best effort checks don’t fail if absent).
- PM2 **may** be present; commands are no-ops if not installed.
- You have `sudo` privileges and can reload/restart NGINX safely.
- TLS internal upstreams may need `proxy_ssl_server_name on;` (verification strictly controlled by your trust store).
- Kernel sysctl one-liners are safe defaults for general workloads (no impact on correctness, only capacity).

---

🧙🏾‍♂️: This runbook gives you a decisive, copy‑pastable path to **diagnose** and **stop** intermittent 502s, with immediate mitigations and clear verification steps.