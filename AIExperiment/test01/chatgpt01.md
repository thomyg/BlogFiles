🧙🏾‍♂️: Alright Thomy—here’s a complete, production-ready runbook to kill intermittent 502s on an NGINX → Node.js stack on Linux. No questions, just decisive steps.

---

# 1) Decision Tree (crisp)

**A. Confirm 502 pattern & where it originates**

* A1. 502s present in NGINX `access.log`? → **Yes** → A2. **No** → issue likely upstream client/CDN → skip to **H**.
* A2. Check `error.log` for upstream markers (`upstream timed out`, `no live upstreams`, `connect() failed`, `recv() failed`) →

  * If **timeout** → go **B**.
  * If **connect() failed / refused** → go **C**.
  * If **header/body too big** → go **D**.
  * If **upstream sent no valid HTTP** (bad gateway) → go **E**.
  * If **SSL to upstream** errors → go **F**.
  * If **no live upstreams / DNS** → go **G**.

**B. Timeouts (read/send/connect)**

* B1. Is Node CPU-bound or event-loop blocked? → If high CPU / long GC pauses → **B2** else **B3**.
* B2. Optimize/mitigate (timeouts, keepalive, workers) → **Actions B**.
* B3. Network/connectivity slow? → **Actions B-net**.

**C. Connect() failed / connection refused / reset**

* C1. Is Node process up and listening on expected port? → **Actions C**.
* C2. LISTEN backlog or FD exhaustion? → **Actions C-limits**.

**D. Header/body too large**

* D1. Requests with large headers (cookies) or big uploads? → **Actions D**.

**E. “upstream sent no valid HTTP response”**

* E1. Node writing before headers / crashing mid-response / HTTP/1.1 framing? → **Actions E**.

**F. Upstream SSL/TLS errors**

* F1. Cert/ALPN/protocol mismatch or self-signed without `proxy_ssl_verify off`? → **Actions F**.

**G. DNS / upstream selection**

* G1. NGINX resolver cache or DNS flaps? Upstream by hostname? → **Actions G**.

**H. Outside NGINX (CDN/WAF/LB)**

* H1. Compare edge vs origin logs; if mismatch → disable edge briefly or bypass → **Actions H**.

---

# 2) Exact Commands & Log Locations (per branch)

> **Common paths (Debian/Ubuntu/CentOS/RHEL):**
> NGINX conf: `/etc/nginx/nginx.conf`, vhosts: `/etc/nginx/sites-enabled/*` or `/etc/nginx/conf.d/*.conf`
> Logs: `/var/log/nginx/access.log`, `/var/log/nginx/error.log`
> Systemd: `systemctl status <svc>`, `journalctl -u <svc> -S -2h`
> Node service name examples: `node`, `node-app`, `pm2-<user>`

**Baseline triage (run first):**

```bash
# Show recent 502s with upstream info
sudo awk '$9==502 {print $0}' /var/log/nginx/access.log | tail -n 50
sudo tail -n 200 /var/log/nginx/error.log

# NGINX health & build
sudo nginx -t && nginx -V
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx -S -2h --no-pager

# Node health
pgrep -a node || true
sudo ss -ltnp | awk 'NR==1 || /:3000|:8080|:4000/'   # adjust if your app port differs
sudo systemctl status node || true
sudo systemctl status node-app || true
sudo journalctl -u node -S -2h --no-pager || true
sudo journalctl -u node-app -S -2h --no-pager || true
pm2 status || true
pm2 logs --lines 200 || true
```

### Branch B — Timeouts

**Detect event-loop/CPU/GC pressure, slow handlers, slow upstream dependencies**

```bash
# Is NGINX reporting timeouts?
sudo egrep -i "timed out|upstream.*timeout" /var/log/nginx/error.log | tail -n 100

# Server load & CPU steal/waits
uptime
top -H -b -n1 | head -n 50
pidstat -tl 1 5 | sed -n '1,120p'    # if sysstat installed

# Node event loop lag (quick probe if using pm2)
pm2 monit || true

# Socket backlog / retransmits / latency hints
sudo ss -s
sudo ss -ti state established '( sport = :3000 or sport = :8080 )' | head -n 50

# System pressure / OOM
dmesg -T | egrep -i "oom|kill" || true
```

**Actions B (timeouts mitigation in NGINX):**

```bash
# Drop in safe timeout/keepalive overrides
sudo tee /etc/nginx/conf.d/99-timeouts.conf >/dev/null <<'EOF'
proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
send_timeout 60s;
proxy_http_version 1.1;
proxy_set_header Connection "";
keepalive_timeout 65s;
keepalive_requests 1000;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**Actions B-net (if network slowness suspected):**

```bash
# Enable upstream keepalive to reduce TCP setup cost (if using upstream blocks)
sudo sed -i '/upstream .*{/,/}/ s/}/    keepalive 64;\n}/' /etc/nginx/sites-enabled/* 2>/dev/null || true
sudo nginx -t && sudo systemctl reload nginx
```

### Branch C — connect() failed / refused / reset

**Diagnostics**

```bash
# Is Node listening where NGINX points?
sudo grep -R "proxy_pass" -n /etc/nginx/{sites-enabled,conf.d} | sed -n '1,120p'
sudo ss -ltnp | grep -E ":3000|:8080|:4000|:9000" || echo "No app listener found"

# Process restarts or crashes
sudo journalctl -u node -S -12h --no-pager || true
sudo journalctl -u node-app -S -12h --no-pager || true
pm2 logs --lines 200 || true

# FD exhaustion / ulimit
ulimit -n
cat /proc/sys/fs/file-nr
sudo lsof -p $(pgrep -n node) | wc -l
```

**Actions C (bring service up / align port):**

```bash
# If service down, start it
sudo systemctl start node || sudo systemctl start node-app || true

# If NGINX points to wrong port, hotfix via override (adjust 3000 accordingly)
sudo sed -i 's|proxy_pass http://127\.0\.0\.1:.*;|proxy_pass http://127.0.0.1:3000;|g' /etc/nginx/sites-enabled/* 2>/dev/null || true
sudo nginx -t && sudo systemctl reload nginx
```

**Actions C-limits (increase file descriptors safely):**

```bash
# Systemd override for Node service (replace node-app if different)
sudo systemctl edit node-app <<'EOF'
[Service]
LimitNOFILE=1048576
EOF
sudo systemctl daemon-reload && sudo systemctl restart node-app

# Kernel & NGINX worker connections
echo 'fs.file-max=2097152' | sudo tee /etc/sysctl.d/99-fd.conf
sudo sysctl --system
sudo sed -i 's/worker_connections .*/worker_connections 65535;/' /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
```

### Branch D — header/body too large

**Diagnostics**

```bash
sudo egrep -i "upstream sent too big header|header too large|client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

**Actions D (increase buffers & body size):**

```bash
sudo tee /etc/nginx/conf.d/99-buffers.conf >/dev/null <<'EOF'
client_max_body_size 50m;
proxy_buffers 16 32k;
proxy_buffer_size 64k;
large_client_header_buffers 8 32k;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

### Branch E — “upstream sent no valid HTTP response”

**Diagnostics**

```bash
# Look for Node stack traces mid-response
pm2 logs --lines 300 || true
sudo journalctl -u node-app -S -2h --no-pager || true

# Verify backend speaks HTTP/1.1 and not raw TCP or HTTP/2 prior knowledge
sudo curl -v http://127.0.0.1:3000/ -H 'Connection: close' --max-time 5
```

**Actions E (normalize proxying):**

```bash
sudo tee /etc/nginx/conf.d/99-proxy-headers.conf >/dev/null <<'EOF'
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Connection "";
EOF
sudo nginx -t && sudo systemctl reload nginx
```

### Branch F — upstream SSL/TLS errors

**Diagnostics**

```bash
sudo egrep -i "proxy_ssl|SSL|certificate" /var/log/nginx/error.log | tail -n 100
# Test upstream TLS directly
openssl s_client -connect 127.0.0.1:3001 -servername example.local -alpn http/1.1 -brief </dev/null
```

**Actions F (safe defaults for internal TLS):**

```bash
sudo tee /etc/nginx/conf.d/99-proxy-ssl.conf >/dev/null <<'EOF'
proxy_ssl_server_name on;
proxy_ssl_session_reuse on;
# If using internal/self-signed certs ONLY; otherwise keep verify ON.
proxy_ssl_verify off;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

### Branch G — DNS / upstream selection

**Diagnostics**

```bash
# Is proxy_pass using hostname?
sudo grep -R "proxy_pass http" -n /etc/nginx/{sites-enabled,conf.d} | grep -v 127.0.0.1 || true

# Resolver in NGINX?
sudo grep -R "resolver " -n /etc/nginx/nginx.conf /etc/nginx/conf.d || true

# Check OS DNS
cat /etc/resolv.conf
sudo systemd-resolve --status 2>/dev/null | sed -n '1,120p' || true
```

**Actions G (stable resolver + cache TTL):**

```bash
# Add a robust resolver (Cloudflare + Google) with TTL
sudo tee /etc/nginx/conf.d/99-resolver.conf >/dev/null <<'EOF'
resolver 1.1.1.1 8.8.8.8 valid=60s ipv6=off;
resolver_timeout 5s;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

### Branch H — CDN/WAF/LB mismatch

**Diagnostics**

```bash
# Compare counts at edge vs origin (replace paths if needed)
echo "Origin 502s:" && sudo awk '$9==502{c++} END{print c+0}' /var/log/nginx/access.log
# Pull CDN logs (provider CLI) – if unavailable, test direct origin:
curl -I https://origin.yourdomain.tld/healthz --resolve origin.yourdomain.tld:443:ORIGIN_IP --max-time 5 -k
```

**Actions H (bypass to isolate):**

```bash
# Temporarily set cache-bypass for troubleshooting (won't disrupt regular users heavily)
sudo tee /etc/nginx/conf.d/99-bypass-debug.conf >/dev/null <<'EOF'
map $http_x_debug_bypass $bypass {
  default 0;
  1 1;
}
add_header X-Debug-Bypass $bypass;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

---

# 3) Probable Root Causes (ranked with confidence)

1. **Backend slowness causing upstream timeouts (event loop blocking, heavy GC, slow DB calls)** — **35%**
2. **Node crashes/restarts or not listening on expected port** — **20%**
3. **File descriptor/socket exhaustion (ulimit/worker_connections/backlog)** — **12%**
4. **Keep-alive/proxy headers misconfig causing connection churn** — **10%**
5. **Headers/body exceed NGINX defaults (cookies/uploads)** — **8%**
6. **TLS to upstream misconfigured (SNI/verify/ALPN)** — **5%**
7. **DNS flapping or missing resolver in NGINX** — **3%**
8. **Edge/CDN/WAF differences vs origin responses** — **2%**
9. **NGINX bug or rare kernel issue** — **1%**
10. **Other (ephemeral ports/conntrack saturation, packet loss)** — **4%**

---

# 4) One-liner Safe Mitigations (per cause)

> These are **safe**, reversible hotfixes you can apply immediately; follow with deeper fixes in code/infrastructure.

1. **Backend slowness / timeouts**

```bash
sudo tee /etc/nginx/conf.d/99-timeouts.conf >/dev/null <<'EOF'
proxy_connect_timeout 5s; proxy_send_timeout 60s; proxy_read_timeout 60s; proxy_http_version 1.1; proxy_set_header Connection "";
EOF
sudo nginx -t && sudo systemctl reload nginx
```

2. **Node crashes / not listening**

```bash
sudo systemctl restart node-app || pm2 restart all
```

3. **FD/socket exhaustion**

```bash
sudo systemctl edit node-app <<< $'[Service]\nLimitNOFILE=1048576' && sudo systemctl daemon-reload && sudo systemctl restart node-app
```

4. **Keep-alive / header normalization**

```bash
sudo tee /etc/nginx/conf.d/99-proxy-headers.conf >/dev/null <<'EOF'
proxy_http_version 1.1; proxy_set_header Connection ""; proxy_set_header Host $host; proxy_set_header X-Forwarded-Proto $scheme;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

5. **Large headers/body**

```bash
sudo tee /etc/nginx/conf.d/99-buffers.conf >/dev/null <<'EOF'
client_max_body_size 50m; large_client_header_buffers 8 32k; proxy_buffers 16 32k; proxy_buffer_size 64k;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

6. **Upstream TLS mismatch**

```bash
sudo tee /etc/nginx/conf.d/99-proxy-ssl.conf >/dev/null <<'EOF'
proxy_ssl_server_name on; proxy_ssl_session_reuse on; proxy_ssl_verify off;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

7. **DNS instability**

```bash
sudo tee /etc/nginx/conf.d/99-resolver.conf >/dev/null <<'EOF'
resolver 1.1.1.1 8.8.8.8 valid=60s ipv6=off; resolver_timeout 5s;
EOF
sudo nginx -t && sudo systemctl reload nginx
```

8. **CDN/WAF mismatch**

```bash
# Temporarily bypass edge by testing direct origin with /etc/hosts mapping
sudo bash -c 'echo "ORIGIN_IP origin.yourdomain.tld" >> /etc/hosts' && curl -I https://origin.yourdomain.tld/healthz --max-time 5 -k
```

9. **Rare NGINX bug/kern corner**

```bash
sudo apt-get update && sudo apt-get install --only-upgrade nginx || sudo yum update -y nginx
```

10. **Ephemeral ports/conntrack saturation (conservative bump)**

```bash
echo -e "net.ipv4.ip_local_port_range=10240 65535\nnet.netfilter.nf_conntrack_max=262144" | sudo tee /etc/sysctl.d/99-net.conf && sudo sysctl --system
```

---

# 5) Assumptions Made

* Linux systemd host with root/sudo access; NGINX installed from distro packages; logs in `/var/log/nginx/`.
* Node app runs as a systemd service named `node-app` or via `pm2`. Replace names/ports where noted.
* Reverse proxy is simple HTTP/1.1 to upstream on localhost or RFC1918; HTTPS to upstream covered in branch F.
* Safe defaults chosen to mitigate without materially weakening security; **`proxy_ssl_verify off`** is only intended for **internal** TLS during triage—re-enable with proper CA trust thereafter.
* Buffer sizes and timeouts selected conservatively to avoid memory bloat while eliminating common 502 triggers.
* No external CDN/WAF specifics known; provided neutral bypass and comparison steps.
* Commands are Debian/Ubuntu oriented but compatible with RHEL/CentOS with minor package manager changes.

---

🧙🏾‍♂️: To proceed, start with the **Baseline triage**, follow the **Decision Tree**, apply the matching **One-liner**, and then observe logs for 15–30 minutes to confirm 502s stop.

🙂: Want me to package these into a single bash script that runs the checks, extracts indicators, and suggests the branch automatically?
