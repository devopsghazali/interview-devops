# 🚨 DevOps Runbook: App Slowness Troubleshooting (Step-by-Step)

---

## 🧠 Scenario

User reports:

> "App slow chal rahi hai"

👉 Goal: Root cause identify + fix (NOT just restart)

---

# 🔍 Step 1: Confirm Issue (Monitoring)

```bash
top
htop
```

Check:

* CPU usage < 70% ✅
* Memory available ✅

---

# 🌐 Step 2: Network Latency Check

```bash
ping google.com
```

Check:

* High latency? ❌
* Packet loss? ❌

```bash
traceroute google.com
```

👉 Ensure no network delay

---

# 📡 Step 3: Bandwidth Check

```bash
iftop
```

OR

```bash
nload
```

Check:

* Bandwidth low usage → NOT bottleneck
* If high → investigate IPs

---

# 📜 Step 4: Application Logs (IMPORTANT)

```bash
tail -f /var/log/app.log
```

Look for:

* Errors
* Timeouts
* Slow queries

---

# 🔗 Step 5: Active Connections Check

```bash
netstat -antp
```

### Count total connections

```bash
netstat -ant | wc -l
```

### Check specific port (e.g., 80)

```bash
netstat -ant | grep :80
```

---

# 🚨 Step 6: Identify High Connections

```bash
netstat -ant | grep :80 | wc -l
```

👉 If very high (e.g., 1000+):

* Possible overload
* Bot / attack

---

# 🔎 Step 7: Identify Top IPs

```bash
netstat -ant | grep :80 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

👉 Output shows:

* Which IP is making most connections

---

# ❌ Step 8: Block / Close Unnecessary Connections

## Option 1: Block IP (iptables)

```bash
sudo iptables -A INPUT -s <IP> -j DROP
```

## Option 2: Kill process (if local issue)

```bash
ps aux | grep <process>
kill -9 <PID>
```

---

# 🔄 Step 9: Check for TIME_WAIT flood

```bash
netstat -ant | grep TIME_WAIT | wc -l
```

👉 If too high:

* Tune kernel settings

---

# ⚙️ Step 10: Modern Alternative (ss command)

```bash
ss -s
```

```bash
ss -antp
```

---

# 🎯 Final Decision Tree

```bash
CPU high?        → Optimize / scale
Memory full?     → Fix leak / scale
Network issue?   → Fix routing
Bandwidth high?  → Investigate traffic
Connections high?→ Block IP / scale
Logs error?      → Fix code/DB
```

---

# 💡 Pro DevOps Tips

* Never restart blindly ❌
* Always find root cause ✅
* Keep monitoring dashboards ready

---

# 🧾 Summary

> Slow app debugging = Infra → Network → Bandwidth → Logs → Connections → Fix

---

🔥 This is real-world production troubleshooting flow used by DevOps engineers.
