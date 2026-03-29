# 🧾 Logrotate Deep Dive (DevOps Level Notes)

---

## 🧠 Problem Statement
- Logs continuously grow (e.g., access.log)
- Disk fills up → server slow/crash
- Manual cleanup (rm) is dangerous ❌

---

## ❌ Why `rm access.log` is BAD
- File gets deleted from directory
- But process (nginx/app) still holds it open
- Disk space **NOT freed**
- Hidden usage → dangerous situation

---

## ✅ Correct Quick Fix
```bash
> access.log
# OR
truncate -s 0 access.log
```

✔ File stays same  
✔ Process unaffected  
✔ Space freed  

---

## 🏆 Permanent Solution: Logrotate

### 📂 Config File Location
```bash
/etc/logrotate.d/myapp
```

---

## ✅ Basic Config Example
```conf
/var/log/myapp/access.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
```

---

## 🔍 Line-by-Line Explanation

### 🔁 daily
- Rotate logs every day

### 🔢 rotate 7
- Keep last 7 log files
- Older ones → deleted

### 🗜 compress
- Old logs → `.gz`
- Saves disk space

### ❌ missingok
- If file missing → ignore (no error)

### 🚫 notifempty
- Skip rotation if file is empty

### 🔥 copytruncate (IMPORTANT)
- Copy current log → new file
- Empty original file

✔ App keeps writing  
✔ No restart needed  

---

## ⚙️ Internals: How Rotation Works

---

### 🧩 METHOD 1: copytruncate (Simple)

#### Flow:
1. access.log → copied → access.log.1
2. access.log → truncated (empty)

#### Key Point:
- File **same rehti hai**
- Process ko kuch pata nahi chalta

✔ Safe for simple apps  
⚠️ Slight data loss risk (during copy)

---

### 🧩 METHOD 2: Rename + Signal (PRO LEVEL)

#### Flow:
1. access.log → renamed → access.log.1
2. New file created → access.log
3. App still writing to old file ❌

#### Solution:
```bash
kill -USR1 <process-id>
# or
systemctl reload nginx
```

#### Result:
- App reopens file
- Starts writing to new log

✔ No data loss  
✔ Production standard  

---

## 🔁 Example (Production Style)

```conf
/var/log/nginx/access.log {
    daily
    rotate 7
    compress
    postrotate
        systemctl reload nginx
    endscript
}
```

---

## ⚡ Logrotate Runs Automatically

- Controlled by cron:
```bash
/etc/cron.daily/logrotate
```

---

## 🧪 Manual Testing

```bash
sudo logrotate -f /etc/logrotate.d/myapp
```

---

## 🔍 Verify Output

```bash
ls /var/log/myapp/
```

Expected:
```
access.log
access.log.1
access.log.2.gz
```

---

## 🧠 How Program Knows Which File to Write?

### Case 1: copytruncate
- File same rehti hai
- Program already using it
- No change needed

---

### Case 2: rename method
- Program still writing to old file
- Signal/reload required
- Then switches to new file

---

## ⚔️ Comparison

| Feature | copytruncate | signal/reload |
|--------|------------|--------------|
| Easy | ✅ | ❌ |
| Data safety | ⚠️ | ✅ |
| Restart needed | ❌ | ✅ |
| Production use | ❌ limited | ✅ best |

---

## 🚨 Common Mistakes

❌ Using `rm` on active logs  
❌ Not using logging (`>> file 2>&1`)  
❌ Forgetting `copytruncate`  
❌ Not reloading service in rename method  

---

## 🎯 Final DevOps Philosophy

- ❌ rm = dangerous  
- ✅ truncate = quick fix  
- 🏆 logrotate = permanent solution  

---

## 🚀 One-Line Summary

**Logrotate automatically manages log lifecycle — rotate, compress, and clean — without breaking running applications**
