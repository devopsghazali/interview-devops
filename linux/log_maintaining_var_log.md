# 🧾 LOGROTATE SETUP for `/var/log/mylog/` (Production + Beginner Friendly)

---

# 🚀 SUMMARY (PEHLE HI SAMJHO)

Ye setup automatically karega:

✔ Har din logs check  
✔ 7 din purane logs compress (.gz)  
✔ 30 din se purane logs delete  
✔ System level automation (cron manually likhne ki zarurat nahi)

👉 Matlab: disk full nahi hogi + logs automatically manage honge

---

# 🧠 INTRODUCTION

Linux servers me logs continuously grow karte hain.

Agar control na ho:

❌ /var/log full ho jata hai  
❌ system slow ho sakta hai  
❌ services crash kar sakti hain  

Isliye DevOps me use hota hai:

👉 **logrotate (industry standard tool)**

---

# 📁 USE CASE

Folder:

/var/log/mylog/

Hume chahiye:

- 7 days → compress
- 30 days → delete
- automatic daily execution

---

# ⚙️ LOGROTATE CONFIG (MAIN FILE)

File location:

/etc/logrotate.d/mylog

---

# 📜 CONFIG CONTENT

/var/log/mylog/*.log {
    daily
    rotate 30
    maxage 30
    missingok
    notifempty
    compress
    delaycompress
    dateext

    create 0640 root root

    sharedscripts

    postrotate
        echo "Log rotation completed for mylog" > /var/log/mylog/rotation.log
    endscript
}

---

# 🧠 ADVANCED LINE-BY-LINE EXPLANATION

---

## 📅 daily
👉 Har din logrotate run hoga

✔ automatic scheduling  
✔ manual cron ki need nahi

---

## 🔁 rotate 30
👉 30 backup files tak rakhega

✔ approx 30 days history maintain  
✔ purane logs overwrite/delete cycle me jayenge

---

## 🧹 maxage 30
👉 30 din se purani files delete

✔ disk automatically clean rahegi

---

## 📦 compress
👉 rotated logs ko gzip karega

Example:

app.log → app.log.gz

✔ disk space save hota hai

---

## 💤 delaycompress
👉 latest rotated file ko immediately compress nahi karta

✔ safety for active logs  
✔ corruption risk avoid hota hai

---

## 📅 dateext
👉 filename me date add karega

Example:

app.log-2026-03-29.gz

✔ tracking easy hoti hai

---

## ⚠️ missingok
👉 agar log file nahi mili to error nahi dega

✔ safe execution

---

## 🧾 notifempty
👉 empty files ko rotate nahi karega

✔ useless processing avoid

---

## 📂 create 0640 root root
👉 new log file permissions set karta hai

✔ security maintained  
✔ proper ownership control

---

## 🔧 sharedscripts + postrotate

Rotation ke baad run hota hai:

echo "Log rotation completed for mylog" > /var/log/mylog/rotation.log

✔ monitoring + debugging helpful

---

# 🧪 TESTING (IMPORTANT)

## 🔍 Dry run (safe check)

sudo logrotate -d /etc/logrotate.d/mylog

👉 kuch bhi delete nahi hoga, sirf simulation

---

## ⚡ Force run (test)

sudo logrotate -f /etc/logrotate.d/mylog

👉 immediately rotation test karega

---

# 🧠 REAL DEVOPS THINKING

Logrotate ka fayda sirf cleanup nahi hai:

✔ system stable rehta hai  
✔ disk full issues avoid hote hain  
✔ logs structured rehte hain  
✔ production reliability improve hoti hai  

---

# 🔥 BEFORE vs AFTER

## ❌ Without logrotate
- manual scripts needed  
- cron errors possible  
- disk full risk  
- inconsistent cleanup  

## ✅ With logrotate
- automatic system managed  
- production standard  
- stable + safe  
- widely used in Nginx, Apache, Syslog systems  

---

# 📊 SIMPLE FLOW

Logs create hote hain
    ↓
Logrotate daily run hota hai
    ↓
7 days → compress
    ↓
30 days → delete
    ↓
System clean + stable

---

# 🎯 FINAL NOTE

👉 Logrotate is not optional in real systems  
👉 It is a MUST-HAVE DevOps tool  

Agar tum next level jana chahte ho to:

- log monitoring (alerts)
- ELK / Loki integration
- AWS S3 log archiving

ye sab bhi production systems me use hota hai
