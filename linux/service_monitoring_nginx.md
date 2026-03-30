# 🛡️ Linux Service Monitoring & systemctl Guide (Beginner to Practical)

## 📌 Overview

Is document mein aap seekhenge:

* Linux services kya hoti hain
* systemctl kya hai aur kaise kaam karta hai
* Application vs Service difference
* Bash script se multiple services monitor karna
* Auto restart + logging concept

---

# 🧠 1. Linux Service kya hoti hai?

Linux mein **service (daemon)** ek background process hota hai jo continuously chalta rehta hai.

### 🔹 Examples:

* nginx → web server service
* mysql → database service
* redis → cache service
* ssh → remote login service

👉 Ye services user ke bina bhi chalti rehti hain.

---

# ⚙️ 2. systemctl kya hai?

`systemctl` ek command-line tool hai jo **systemd system manager** ko control karta hai.

### 🔹 Common commands:

```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
systemctl is-active nginx
```

👉 Simple meaning:

> systemctl = Linux ka remote control for services

---

# 🔥 3. Kyun systemctl use hota hai?

Modern Linux systems mein **systemd** hota hai jo:

* Boot ke time services start karta hai
* Crash hone par auto restart kar sakta hai
* Dependencies manage karta hai
* Logs maintain karta hai

👉 Is wajah se services manually run nahi karni padti

---

# 🧩 4. Application vs Service Difference

## 🖥️ Application

* User run karta hai
* GUI ya CLI hota hai
* Manually start/stop hota hai

### Examples:

* Chrome browser
* VS Code
* Python script

---

## 🛠️ Service

* Background mein chalti hai
* System automatically manage karta hai
* Boot ke time start hoti hai

### Examples:

* nginx
* mysql
* redis

---

## ⚡ Simple Analogy

| Type        | Example                        |
| ----------- | ------------------------------ |
| Application | Car (driver chalata hai)       |
| Service     | Engine (always running system) |

---

# 🧪 5. Single Service Monitoring Script

```bash
#!/bin/bash

SERVICE="nginx"
LOG_FILE="/var/log/service_health.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - [$SERVICE] $1" >> $LOG_FILE
}

systemctl is-active --quiet $SERVICE

if [ $? -ne 0 ]; then
    log_message "❌ Service DOWN"
    systemctl restart $SERVICE
    log_message "🔄 Service restarted"
else
    log_message "✅ Service RUNNING"
fi
```

---

# 🔁 6. Multi-Service Monitoring Script

```bash
#!/bin/bash

LOG_FILE="/var/log/service_health.log"
SERVICES=("nginx" "mysql" "redis")

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - [$1] $2" >> $LOG_FILE
}

for SERVICE in "${SERVICES[@]}"
do
    systemctl is-active --quiet $SERVICE

    if [ $? -ne 0 ]; then
        log_message "$SERVICE" "❌ Service DOWN"
        systemctl restart $SERVICE
        log_message "$SERVICE" "🔄 Service restarted"
    else
        log_message "$SERVICE" "✅ Service RUNNING"
    fi
done
```

---

# ⏱️ 7. Automation (Cron Job)

Script ko auto run karne ke liye:

```bash
crontab -e
```

Add this:

```bash
*/2 * * * * /bin/bash /root/check.sh
```

👉 Matlab har 2 minute mein check hoga

---

# 📊 8. Logging System

Logs store hoti hain:

```
/var/log/service_health.log
```

### Example logs:

```
2026-03-30 12:10:01 - [nginx] ✅ Service RUNNING
2026-03-30 12:10:01 - [mysql] ❌ Service DOWN
2026-03-30 12:10:01 - [mysql] 🔄 Service restarted
```

---

# 🚀 9. Key Concepts Summary

### ✔ Service

Background system process

### ✔ systemctl

Service control tool

### ✔ systemd

System manager jo services ko handle karta hai

### ✔ Script monitoring

Automation jo service crash detect karta hai

---

# 🔥 Final Summary

👉 Linux services system background processes hoti hain
👉 systemctl unko control karta hai
👉 systemd unko manage karta hai
👉 Bash scripts se hum monitoring + auto-restart kar sakte hain

---

# 🚀 Next Level (Optional Learning)

Agar aap chaho to next topics:

* Custom systemd service banana
* Prometheus + Grafana monitoring
* Telegram alert system
* Load balancer failover automation
