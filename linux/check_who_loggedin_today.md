# 👥 Linux Users, Groups & Permissions (Short Course)

## 📌 Overview

Is short course mein aap seekhenge:

* Linux users kya hote hain
* Groups ka concept
* Aaj kitne users login hue (tracking)
* `id`, `who`, `last` commands
* File ownership (`chown`, `chmod`)
* Real DevOps usage

---

# 👤 1. Linux User Kya Hota Hai?

Linux mein har kaam kisi **user** ke under hota hai.

### 🔹 Types of Users:

| Type        | Meaning                         |
| ----------- | ------------------------------- |
| root        | Superuser (poora control)       |
| normal user | limited permissions             |
| system user | services ke liye (nginx, mysql) |

---

# 👥 2. Groups Kya Hote Hain?

Group = users ka collection

👉 Ek group multiple users ko same permissions deta hai.

### 🔹 Example:

* dev team group
* admin group
* docker group

---

# 🧠 3. Current User Check

```bash
whoami
```

👉 Output:

```
ubuntu
```

---

# 🧾 4. User Details Check

```bash
id
```

### 🔹 Output Example:

```
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),27(sudo)
```

## 🧠 Explanation:

| Part   | Meaning                  |
| ------ | ------------------------ |
| uid    | user ID                  |
| gid    | group ID                 |
| groups | user kis groups mein hai |

---

# 📊 5. Aaj Kitne Users Login Hue?

## 🔹 Current logged-in users:

```bash
who
```

👉 Example:

```
ubuntu pts/0 2026-03-30 10:00
root   pts/1 2026-03-30 11:00
```

---

## 🔹 Count logged-in users:

```bash
who | wc -l
```

👉 Output:

```
2
```

---

## 🔥 6. Today Login History (Full)

```bash
last
```

👉 Shows:

* kaun login hua
* kab login hua
* reboot history

---

## 🔥 7. Today specific login filter

```bash
last | grep "$(date '+%b %d')"
```

👉 Sirf aaj ke logins show karega

---

# 📁 8. File Ownership Concept

Linux mein har file ka:

* owner (user)
* group

```bash
ls -l
```

### 🔹 Output Example:

```
-rw-r--r-- 1 root root 1200 file.txt
```

---

## 🧠 Breakdown:

| Part       | Meaning    |
| ---------- | ---------- |
| root (1st) | owner user |
| root (2nd) | group      |

---

# 🔄 9. chown Command (Ownership Change)

## 🔹 Basic syntax:

```bash
chown user file
```

---

## 🔹 Example:

```bash
chown ubuntu file.txt
```

👉 file ka owner ubuntu ho jayega

---

## 🔥 Change user + group:

```bash
chown ubuntu:dev file.txt
```

👉 user = ubuntu
👉 group = dev

---

# 🧠 10. chmod (Permissions)

```bash
chmod 755 file.txt
```

## 🔹 Meaning:

| Digit | Meaning                |
| ----- | ---------------------- |
| 7     | read + write + execute |
| 5     | read + execute         |
| 0     | no permission          |

---

# 👥 11. Group Commands

## 🔹 Check groups of user:

```bash
groups
```

---

## 🔹 Add user to group:

```bash
usermod -aG docker ubuntu
```

---

# 📊 12. Real DevOps Use Case

👉 Example:

* nginx service runs under `www-data`
* logs owned by root
* developers in dev group

---

# ⚡ 13. Security Concept

| Concept        | Meaning                   |
| -------------- | ------------------------- |
| user isolation | har user alag permissions |
| group access   | shared access control     |
| root access    | full system control       |

---

# 🧠 14. Key Summary

### 👤 User

Individual account

### 👥 Group

Users ka collection

### 📁 Ownership

Files kis user/group ke hain

### 🔄 chown

Ownership change command

### 🔐 chmod

Permissions control

---

# 🚀 Final Summary

👉 Linux system users + groups pe based hota hai
👉 Har file ka owner aur group hota hai
👉 `who`, `last` se login tracking hoti hai
👉 `chown` ownership change karta hai
👉 `chmod` permissions control karta hai

---

# 🔥 Next Level (Optional)

Agar aap chaho to main bana sakta hoon:

* Linux permission octal deep dive
* ACL (advanced permissions)
* sudoers file system
* real multi-user server architecture
