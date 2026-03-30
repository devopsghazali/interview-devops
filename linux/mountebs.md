# 💾 Linux Disk Mounting Deep Dive (Beginner → DevOps)

## 📌 Overview

Is guide mein aap seekhenge:

* Disk, partition aur mounting ka concept
* Temporary vs Permanent mount
* Filesystem (ext4) kya hota hai
* `/etc/fstab` ka real use
* UUID kyun important hai
* Real DevOps best practices

---

# 🧠 1. Disk & Partition Concept

## 🔹 Disk kya hoti hai?

👉 Physical storage device (HDD/SSD)

Example:

```
/dev/xvdb
```

---

## 🔹 Partition kya hota hai?

👉 Disk ka logical tukda

Example:

```
/dev/xvdb1
```

👉 Ek disk ke multiple partitions ho sakte hain

---

# 🔍 2. Disk Check Karna

```bash
lsblk
```

## 🧠 Output samjho:

```
xvdb
 └─xvdb1
```

👉 xvdb = disk
👉 xvdb1 = partition (isko mount karna hai)

---

# 🔐 3. Mounting Kya Hota Hai?

👉 Mounting = disk ko ek folder ke saath attach karna

### 🧠 Analogy:

* Disk = tijori
* Folder = darwaza
* Mount = tijori ko darwaze se jodna

👉 bina mount ke data visible nahi hota

---

# 📁 4. Step 1: Mount Point Banana

```bash
sudo mkdir /mnt/mydata
```

👉 Ye wo folder hai jahan disk ka data dikhega

---

# 🧱 5. Step 2: Filesystem Banana (Formatting)

```bash
sudo mkfs.ext4 /dev/xvdb1
```

## ⚠️ Warning:

👉 Isse pura data delete ho jayega

---

## 🧠 Filesystem kya hota hai?

👉 Data ko organize karne ka tareeka

### Examples:

* ext4 (Linux best)
* xfs
* ntfs (Windows)

---

# 🔁 6. Temporary Mount

```bash
sudo mount /dev/xvdb1 /mnt/mydata
```

## 🔍 Check:

```bash
df -h
```

👉 reboot ke baad mount hat jayega

---

# 🔒 7. Permanent Mount (Production Level)

👉 `/etc/fstab` use hota hai

---

## 🔹 Step 1: UUID nikalo

```bash
sudo blkid /dev/xvdb1
```

👉 Output:

```
UUID="abcd-1234"
```

---

## 🔹 Step 2: fstab edit karo

```bash
sudo nano /etc/fstab
```

---

## 🔹 Step 3: Entry add karo

```
UUID=abcd-1234  /mnt/mydata  ext4  defaults  0  2
```

---

## 🧠 Breakdown:

| Field       | Meaning          |
| ----------- | ---------------- |
| UUID        | disk identity    |
| /mnt/mydata | mount point      |
| ext4        | filesystem       |
| defaults    | standard options |
| 0           | backup flag      |
| 2           | fsck order       |

---

## 🔹 Step 4: Test karo

```bash
sudo mount -a
```

👉 error nahi = config correct

---

# 🚨 8. UUID vs Device Name (IMPORTANT)

## ❌ Wrong approach:

```
/dev/xvdb1
```

👉 reboot ke baad change ho sakta hai

---

## ✅ Correct approach:

```
UUID=xxxx
```

👉 permanent aur reliable

---

# 🧠 9. Real DevOps Use Cases

👉 Disk use hoti hai:

* Docker volumes
* Database storage
* Logs (/var/log)
* Application data (/var/www)

---

# ⚡ 10. Common Errors

| Error             | Reason           |
| ----------------- | ---------------- |
| mount failed      | wrong filesystem |
| permission denied | wrong ownership  |
| device not found  | wrong path       |
| boot fail         | fstab error      |

---

# 🛡️ 11. Safety Tips

👉 hamesha test karo:

```bash
mount -a
```

👉 backup lo before editing fstab

---

# 🔥 12. Full Workflow Summary

1. lsblk → disk check
2. mkdir → mount point
3. mkfs → format
4. mount → temporary
5. blkid → UUID
6. fstab → permanent entry
7. mount -a → test

---

# 🚀 Final Summary

👉 Disk directly use nahi hoti, mount karni padti hai
👉 Temporary mount reboot pe chali jati hai
👉 Permanent mount fstab se hoti hai
👉 UUID use karna best practice hai

---

# 🔥 DevOps Insight

👉 Junior: mount command janta hai
👉 Senior: fstab + UUID + failure handling janta hai

---

# 🚀 Next Level (Optional)

* LVM (Logical Volume Management)
* Auto disk resize in cloud
* Mount options tuning (noatime, ro)
* Disk monitoring (df, du, alerts)
