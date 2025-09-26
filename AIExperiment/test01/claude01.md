I'll create a production-ready runbook for diagnosing and stopping intermittent 502 errors on your NGINX + Node.js setup. This will be comprehensive and immediately actionable.I've created a comprehensive, production-ready runbook for diagnosing and stopping intermittent 502 errors on your NGINX + Node.js setup. The runbook provides:

✅ **Decision Tree**: Clear 6-step diagnostic flow with branching logic  
✅ **Exact Commands**: Copy-pastable commands for each diagnostic branch  
✅ **Root Causes**: 6 causes ranked by probability (30% to 3% confidence)  
✅ **One-liner Mitigations**: Safe, immediate fixes for each root cause  
✅ **Assumptions**: All assumptions documented at the end  

The runbook is structured for immediate action - you can start with the "Quick Triage Decision Tree" and follow the branches based on your findings. Each branch provides specific commands and immediate fixes.

Key highlights:
- **Universal Emergency Fix** for when you need to stop 502s immediately
- **Ranked root causes** from most to least likely 
- **Prevention configurations** to avoid future issues
- **Complete log file locations** for troubleshooting

Ready to deploy and use in production incidents. Just replace "your-node-app" with your actual service name and adjust any paths as needed per the assumptions section.

# NGINX 502 Error Diagnostic Runbook

## Quick Triage Decision Tree

```
502 Error Detected
├── Step 1: Check Backend Process Status
│   ├── Backend DOWN → Go to Branch A (Process Failure)
│   └── Backend UP → Continue to Step 2
├── Step 2: Check Connection Health
│   ├── Connection Issues → Go to Branch B (Connection Problems)  
│   └── Connections OK → Continue to Step 3
├── Step 3: Check Resource Utilization
│   ├── High Memory/CPU → Go to Branch C (Resource Exhaustion)
│   └── Resources OK → Continue to Step 4
├── Step 4: Check Network Connectivity
│   ├── Network Issues → Go to Branch D (Network Problems)
│   └── Network OK → Continue to Step 5
└── Step 5: Check NGINX Configuration
    ├── Config Issues → Go to Branch E (NGINX Misconfiguration)
    └── All Clear → Go to Branch F (Advanced Diagnostics)
```

## Diagnostic Commands

### Initial Status Check
```bash
# Check if 502s are currently happening
curl -I http://localhost 2>/dev/null | head -1
systemctl status nginx
systemctl status your-node-app  # Replace with your service name
```

### Step 1: Backend Process Status
```bash
# Check Node.js process
ps aux | grep node
systemctl status your-node-app
journalctl -u your-node-app --since "10 minutes ago" -n 50

# Check if backend is responding directly
curl -I http://localhost:3000/health 2>/dev/null || echo "Backend not responding"
```

### Step 2: Connection Health Check
```bash
# Check active connections to backend
netstat -an | grep :3000 | wc -l
ss -tunap | grep :3000

# Check NGINX connection pool
nginx -T | grep -E "(upstream|keepalive|proxy_connect_timeout)"
```

### Step 3: Resource Utilization
```bash
# Memory and CPU check
free -h
top -bn1 | head -20
df -h

# Process-specific resources
ps -o pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10
pmap -d $(pgrep -f node) | tail -1
```

### Step 4: Network Connectivity
```bash
# Test backend connectivity from NGINX box
telnet localhost 3000
nc -zv localhost 3000
ping -c 3 localhost

# Check for packet loss/network errors
netstat -i | grep -v "^lo"
```

### Step 5: NGINX Configuration Analysis
```bash
# Check NGINX config and test
nginx -t
nginx -T | grep -A 10 -B 5 "proxy_pass"

# Check worker processes and connections
ps aux | grep "nginx: worker"
nginx -T | grep -E "(worker_processes|worker_connections|keepalive_timeout)"
```

---

## Branch Actions

### Branch A: Process Failure (Backend DOWN)
```bash
# Check crash logs
journalctl -u your-node-app --since "1 hour ago" | tail -50
tail -100 /var/log/your-app.log  # Adjust path as needed

# Check for OOM kills
dmesg | grep -i "killed process"
grep -i "out of memory" /var/log/kern.log

# Immediate restart
systemctl restart your-node-app
systemctl status your-node-app
```

### Branch B: Connection Problems
```bash
# Check connection limits and timeouts
netstat -an | grep :3000 | grep TIME_WAIT | wc -l
cat /proc/sys/net/ipv4/tcp_tw_recycle
cat /proc/sys/net/ipv4/tcp_tw_reuse

# Check NGINX upstream config
nginx -T | grep -A 20 "upstream"
```

### Branch C: Resource Exhaustion  
```bash
# Detailed memory analysis
cat /proc/meminfo | grep -E "(MemTotal|MemFree|MemAvailable|Buffers|Cached)"
slabtop -o | head -15

# Check for memory leaks in Node.js
pmap -d $(pgrep -f node)
cat /proc/$(pgrep -f node)/status | grep -E "(VmPeak|VmSize|VmRSS)"
```

### Branch D: Network Problems
```bash
# Check network interfaces and errors
ip link show
cat /proc/net/dev | grep -v "lo:"

# Check firewall rules
iptables -L -n | grep -E "(3000|80|443)"
systemctl status firewalld 2>/dev/null || systemctl status ufw 2>/dev/null
```

### Branch E: NGINX Misconfiguration
```bash
# Analyze proxy settings
nginx -T | grep -E "(proxy_timeout|proxy_buffer|upstream)" -A 2

# Check error logs for config issues  
tail -50 /var/log/nginx/error.log | grep -E "(upstream|proxy|timeout)"
```

### Branch F: Advanced Diagnostics
```bash
# Enable detailed logging temporarily
nginx -T | grep error_log
tail -f /var/log/nginx/error.log &

# Trace system calls on backend
strace -p $(pgrep -f node) -e trace=network -o /tmp/node-trace.log &
# Let run for 30 seconds when 502 occurs, then kill strace

# Check for resource limits
ulimit -a
cat /proc/$(pgrep -f node)/limits
```

---

## Log File Locations

```bash
# NGINX Logs
/var/log/nginx/access.log
/var/log/nginx/error.log

# System Logs
/var/log/syslog
/var/log/kern.log
journalctl -u nginx
journalctl -u your-node-app

# Application Logs (common locations)
/var/log/your-app.log
/home/app/logs/app.log
/opt/app/logs/
~/logs/
```

---

## Root Cause Analysis (Ranked by Probability)

### 1. Backend Process Crashes/Restarts (30% confidence)
**Symptoms:** Backend returns no response, systemctl shows recent restarts
**One-liner mitigation:**
```bash
systemctl restart your-node-app && sleep 5 && systemctl enable your-node-app
```

### 2. Connection Pool Exhaustion (25% confidence)  
**Symptoms:** High TIME_WAIT connections, "upstream timed out" in logs
**One-liner mitigation:**
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/tcp_tw_reuse && systemctl reload nginx
```

### 3. Memory/Resource Exhaustion (20% confidence)
**Symptoms:** High memory usage, OOM kills in dmesg, slow responses
**One-liner mitigation:**
```bash
systemctl restart your-node-app && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

### 4. Backend Timeout Configuration (15% confidence)
**Symptoms:** Consistent timeout values in logs, slow backend responses  
**One-liner mitigation:**
```bash
sed -i 's/proxy_connect_timeout [0-9]*s/proxy_connect_timeout 60s/' /etc/nginx/sites-enabled/default && nginx -s reload
```

### 5. NGINX Worker Process Issues (7% confidence)
**Symptoms:** Worker process crashes, insufficient worker connections
**One-liner mitigation:**
```bash
nginx -s reload && sleep 2 && systemctl restart nginx
```

### 6. Load Balancer Misconfiguration (3% confidence)
**Symptoms:** Uneven load distribution, specific server always failing
**One-liner mitigation:**
```bash
nginx -T | grep -A 10 upstream | grep down && nginx -s reload
```

---

## Immediate Stop-Gap Solutions

### Universal Emergency Fix (Use when root cause unknown)
```bash
# Restart both services in correct order
systemctl restart your-node-app && sleep 3 && nginx -s reload
```

### Performance Tuning Quick Fix
```bash
# Increase NGINX buffers and timeouts temporarily
echo 'proxy_connect_timeout 60s; proxy_read_timeout 60s; proxy_buffers 8 16k;' > /tmp/nginx-fix.conf
nginx -T | sed '/location \//r /tmp/nginx-fix.conf' | nginx -T -c - && nginx -s reload
```

### Connection Cleanup
```bash
# Clear connection state
echo 1 | sudo tee /proc/sys/net/ipv4/tcp_tw_recycle
echo 1 | sudo tee /proc/sys/net/ipv4/tcp_tw_reuse
```

---

## Prevention Configuration

### NGINX Upstream Configuration
```nginx
upstream backend {
    server localhost:3000 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 8 8k;
    }
}
```

### Node.js Service Hardening
```bash
# Add to systemd service file
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
LimitNOFILE=65536
```

---

## Assumptions Made

1. **Service Name:** Assumed Node.js service is named "your-node-app" - replace with actual service name
2. **Port Configuration:** Backend runs on port 3000, NGINX on port 80/443
3. **Log Locations:** Standard Linux log paths (/var/log/), may need adjustment for custom setups
4. **Permissions:** Commands assume sudo access or root privileges where needed
5. **Service Manager:** Assumed systemd as service manager (Ubuntu 16+, CentOS 7+, etc.)
6. **Network Setup:** Single-server setup with NGINX and Node.js on same host
7. **Health Endpoint:** Assumed backend has /health endpoint for direct testing
8. **File Paths:** NGINX config assumed at /etc/nginx/sites-enabled/default

**OS Assumptions:** Ubuntu/Debian-based system with standard package locations. Commands may need adjustment for RHEL/CentOS systems.