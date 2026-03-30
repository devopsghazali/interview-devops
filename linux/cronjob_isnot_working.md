# 🛠️ Cron Job Debugging Guide (Production Scenario)

## 📌 Problem Statement

Cron job जो database backup लेता था, वो अचानक बंद हो गया है।
Crontab में कोई error नहीं दिख रहा।

---

# 🎯 Goal

* Identify करें कि job trigger हो रही है या नहीं
* Root cause ढूँढें
* Fix करें और future में prevent करें

---

# 🧠 Step-by-Step Debugging Process

---

## 🔹 Step 1: Cron Service Check करें

सबसे पहले verify करें कि cron service चल रही है या नहीं:

```bash
systemctl status cron
```

अगर बंद है:

```bash
systemctl start cron
```

---

## 🔹 Step 2: Cron Job Verify करें

```bash
crontab -l
```

Check करें:

* timing सही है?
* script path सही है?

---

## 🔹 Step 3: Logs Check करें (Most Important)

```bash
grep CRON /var/log/syslog
```

या:

```bash
journalctl -u cron
```

👉 इससे पता चलेगा:

* job trigger हो रही है या नहीं

---

## 🔹 Step 4: Script Manually Run करें

```bash
bash /home/user/scripts/backup.sh
```

👉 अगर यहाँ fail:

* issue script में है

---

## 🔹 Step 5: Full Path Use करें

### ❌ गलत:

```bash
backup.sh
./backup.sh
```

### ✅ सही:

```bash
/home/user/scripts/backup.sh
```

👉 Cron relative path नहीं समझता

---

## 🔹 Step 6: Commands का Full Path Use करें

### ❌ गलत:

```bash
mysqldump -u root db > backup.sql
```

### ✅ सही:

```bash
/usr/bin/mysqldump -u root db > /home/user/backup.sql
```

---

## 🔹 Step 7: Output Redirect करें (Debugging Trick 🔥)

```bash
* * * * * /home/user/scripts/backup.sh >> /tmp/cron.log 2>&1
```

Check:

```bash
cat /tmp/cron.log
```

👉 असली error यहीं मिलेगा

---

## 🔹 Step 8: Permissions Check करें

```bash
ls -l /home/user/scripts/backup.sh
```

Executable बनाएं:

```bash
chmod +x /home/user/scripts/backup.sh
```

---

## 🔹 Step 9: Shebang Verify करें

Script की पहली लाइन:

```bash
#!/bin/bash
```

---

## 🔹 Step 10: Environment Variables Issue

👉 Cron का environment limited होता है

Fix:

```bash
export PATH=/usr/bin:/bin:/usr/local/bin
```

---

## 🔹 Step 11: Absolute Paths Use करें

### ❌ गलत:

```bash
backup.sql
```

### ✅ सही:

```bash
/home/user/backup.sql
```

---

## 🔹 Step 12: Time / Timezone Check करें

```bash
timedatectl
```

---

# 💣 Common Root Causes

* PATH missing
* Script executable नहीं
* Relative paths use किए गए
* Environment variables missing
* Output redirect नहीं किया
* Cron service बंद

---

# 🚀 Final Working Cron Example

```bash
0 2 * * * /home/user/scripts/backup.sh >> /var/log/backup.log 2>&1
```

---

# 🔐 Best Practices (Production)

* हमेशा full path use करें
* logs redirect करें
* script manually test करें
* permissions सही रखें
* monitoring setup करें

---

# 🎤 Interview Answer (Short)

> First, I will verify whether the cron service is running and check logs to confirm job execution.
> Then I will manually execute the script to isolate script issues.
> I will ensure absolute paths and proper environment variables are used.
> Finally, I will redirect output to logs for debugging and fix permissions if required.

---

# 🧠 Key Insight

👉 Cron silently fail करता है
👉 Debugging के लिए logs + manual testing जरूरी है

---

# ✅ Conclusion

Cron debugging का golden rule:

👉 **"अगर दिख नहीं रहा, तो log में लिखो"**

---
