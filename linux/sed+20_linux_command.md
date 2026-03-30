# 🐧 Linux Essentials + `sed` Short Course (Interview Ready)

## 📌 Overview

Is short course mein aap seekhenge:

* `sed` ka core concept (replace, delete, in-place edit)
* Real DevOps usage of `sed`
* 20 essential Linux commands (jo har engineer ko aane chahiye)

---

# 🧠 PART 1: `sed` (Stream Editor)

## 🔹 `sed` kya hai?

`sed` ek **stream editor** hai jo text ko read aur modify karta hai bina file open kiye.

👉 One-line:

> sed = non-interactive text editing tool

---

## 🔹 Basic Syntax

```bash
sed 's/old/new/' file
```

### 🧠 Meaning:

* `s` = substitute (replace)
* `old` = jo dhundna hai
* `new` = jo replace karna hai

---

## 🔹 Important Variations

### 1. Single replace

```bash
sed 's/nginx/apache/' file.txt
```

### 2. Global replace

```bash
sed 's/nginx/apache/g' file.txt
```

### 3. File update (IMPORTANT)

```bash
sed -i 's/nginx/apache/g' file.txt
```

👉 `-i` = in-place (file permanently change)

---

## 🔹 Delete lines

```bash
sed '/error/d' file.txt
```

👉 jahan `error` mile, wo line delete

---

## 🔹 Specific line

```bash
sed '2d' file.txt
```

👉 line 2 delete

---

## 🔹 Safe edit (backup)

```bash
sed -i.bak 's/nginx/apache/g' file.txt
```

---

## 🔥 Real DevOps Use

```bash
sed -i 's/listen 80;/listen 8080;/' /etc/nginx/sites-available/default
```

---

## 🧠 `sed` Summary

* replace → `s/old/new/`
* global → `g`
* file update → `-i`
* delete → `/pattern/d`

---

# 🐧 PART 2: Top 20 Linux Commands (Must Know)

---

## 📂 File & Directory

### 1. pwd (current directory)

```bash
pwd
```

### 2. ls (list files)

```bash
ls -l
```

### 3. cd (change directory)

```bash
cd /var/www
```

### 4. mkdir (create folder)

```bash
mkdir test
```

### 5. rm (delete)

```bash
rm -rf file
```

---

## 📄 File Handling

### 6. touch (create file)

```bash
touch file.txt
```

### 7. cp (copy)

```bash
cp file1 file2
```

### 8. mv (move/rename)

```bash
mv old new
```

### 9. cat (view file)

```bash
cat file.txt
```

### 10. less (scroll view)

```bash
less file.txt
```

---

## 🔍 Search & Text

### 11. grep (search text)

```bash
grep nginx file.txt
```

### 12. find (search files)

```bash
find / -name file.txt
```

### 13. wc (count)

```bash
wc -l file.txt
```

---

## ⚙️ System & Process

### 14. top (live processes)

```bash
top
```

### 15. ps (process list)

```bash
ps aux
```

### 16. kill (stop process)

```bash
kill -9 PID
```

---

## 👤 Users & Permissions

### 17. id (user info)

```bash
id
```

### 18. chmod (permissions)

```bash
chmod 755 file
```

### 19. chown (ownership)

```bash
chown user:group file
```

---

## 🌐 Networking & Service

### 20. systemctl (service control)

```bash
systemctl status nginx
```

---

# 🧠 Final Summary

👉 `sed` = text automation tool
👉 `-i` = file update
👉 Linux commands = daily DevOps toolkit

---

# 🚀 Interview Tip

👉 sab yaad mat karo
👉 concepts samjho + daily use karo
👉 real practice = real skill
