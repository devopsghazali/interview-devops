# 🧹 Linux Guide: Find & Delete Large Files (100MB+)

## 📌 Overview

Is guide mein aap seekhenge:

* Linux mein large files kaise find karte hain
* 100MB+ files identify karna
* Safe deletion methods
* Commands ka deep explanation
* Practical use in servers (DevOps perspective)

---

# ⚠️ WARNING (IMPORTANT)

👉 Delete karne se pehle hamesha verify karein
👉 Galat file delete karne se system break ho sakta hai
👉 Production server par dhyaan se use karein

---

# 🔍 1. Find Files Larger Than 100MB

## ✅ Command:

```bash
find / -type f -size +100M
```

## 🧠 Explanation:

| Part        | Meaning                            |
| ----------- | ---------------------------------- |
| find        | Linux search command               |
| /           | root directory (poora system scan) |
| -type f     | sirf files (directories nahi)      |
| -size +100M | 100MB se badi files                |

👉 Result: system mein jitni bhi 100MB+ files hain woh list ho jayengi

---

# 📂 2. Better: Specific Directory Scan

```bash
find /var/log -type f -size +100M
```

## 🧠 Why use this?

* Faster scan
* Safe (poora system scan nahi hota)
* Logs folder target hota hai (mostly heavy files yahin hoti hain)

---

# 📊 3. Human Readable Output (Advanced)

```bash
find /var/log -type f -size +100M -exec ls -lh {} \;
```

## 🧠 Explanation:

| Part   | Meaning                                   |
| ------ | ----------------------------------------- |
| -exec  | har file pe command run karo              |
| ls -lh | size human readable format mein show kare |
| {}     | found file ka placeholder                 |
| ;      | command end signal                        |

👉 Output mein file size MB/GB mein dikhai dega

---

# 🧹 4. Delete Files Larger Than 100MB

## ⚠️ SAFE METHOD (Recommended)

```bash
find /var/log -type f -size +100M -delete
```

## 🧠 Explanation:

| Part    | Meaning            |
| ------- | ------------------ |
| -delete | direct file remove |

👉 WARNING: Ye directly delete karta hai

---

# 🛑 5. SAFE METHOD (Recommended for beginners)

```bash
find /var/log -type f -size +100M -exec rm -i {} \;
```

## 🧠 Explanation:

| Part  | Meaning                                |
| ----- | -------------------------------------- |
| rm -i | interactive delete (confirmation lega) |

👉 Har file delete hone se pehle poochega: yes/no

---

# 🔥 6. Best Production Method (Logging + Safety)

```bash
find /var/log -type f -size +100M -print
```

👉 pehle sirf list dekho
👉 verify karo
👉 phir delete karo

---

# 📦 7. Example Real Use Case (Server Cleanup)

## Step 1: Check large files

```bash
find /var/log -type f -size +100M
```

## Step 2: Confirm manually

```bash
ls -lh /var/log
```

## Step 3: Delete safely

```bash
find /var/log -type f -size +100M -delete
```

---

# ⚡ 8. Bonus: Old Files Delete (7 days se purane)

```bash
find /var/log -type f -mtime +7 -delete
```

## 🧠 Explanation:

| Part      | Meaning               |
| --------- | --------------------- |
| -mtime +7 | 7 din se purani files |

---

# 📊 9. Combine Size + Time Filter

```bash
find /var/log -type f -size +100M -mtime +7
```

👉 100MB se badi AND 7 din purani files

---

# 🧠 10. Key Concepts Summary

### 🔹 find

Linux ka search tool

### 🔹 -type f

Sirf files

### 🔹 -size +100M

100MB se badi files

### 🔹 -delete

Direct remove

### 🔹 -exec

Custom command run karna

---

# 🚀 Final Summary

👉 Linux mein large files system slow kar deti hain
👉 find command se easily detect hoti hain
👉 delete carefully karna chahiye
👉 pehle list karo, phir delete

---

# 🔥 Next Level (Optional)

Agar aap chaho to main bana sakta hoon:

* Auto disk cleanup bash script
* Cron based cleanup system
* Log rotation (logrotate deep dive)
* Disk usage alert system
