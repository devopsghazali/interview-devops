# 💾 EBS Volume Mounting Guide (Linux / Production)

## 📌 Problem Statement

आपके पास एक नया disk (जैसे AWS EBS volume) है।
आपको:

* उसे system में attach करना है
* mount करना है
* और reboot के बाद भी auto mount करवाना है

---

# 🎯 Goal

* Disk को पहचानना
* Format करना
* Mount करना
* Permanent mount सेट करना

---

# 🧠 Step-by-Step Process

---

## 🔹 Step 1: Disk Identify करें

```bash
lsblk
```

👉 Output कुछ ऐसा होगा:

```
sda    8:0    0   20G  0 disk
└─sda1 8:1    0   20G  0 part /
sdb    8:16   0   10G  0 disk   ← नया disk
```

👉 यहाँ `sdb` आपका नया EBS volume है

---

## 🔹 Step 2: Partition बनाएं (Optional but recommended)

```bash
fdisk /dev/sdb
```

👉 अंदर:

* `n` → new partition
* `p` → primary
* `w` → save

👉 Result:

```
/dev/sdb1
```

---

## 🔹 Step 3: Format करें

👉 filesystem बनाना जरूरी है

```bash
mkfs.ext4 /dev/sdb1
```

---

## 🔹 Step 4: Mount Point बनाएं

```bash
mkdir /data
```

---

## 🔹 Step 5: Mount करें (Temporary Mount)

```bash
mount /dev/sdb1 /data
```

👉 Check:

```bash
df -h
```

---

# ⚠️ Important Concept

👉 ये mount **temporary** है
👉 reboot के बाद हट जाएगा ❌

---

# 🔥 Temporary vs Permanent Mount

| Feature       | Temporary Mount | Permanent Mount |
| ------------- | --------------- | --------------- |
| Command       | mount           | /etc/fstab      |
| Reboot के बाद | हट जाता है ❌    | बना रहता है ✅   |
| Use case      | testing         | production      |

---

# 🔹 Step 6: Permanent Mount (Auto Mount after Reboot)

👉 इसके लिए `/etc/fstab` use होता है

---

## 📌 Step 6.1: UUID निकालो

```bash
blkid
```

👉 Output:

```
/dev/sdb1: UUID="abc123" TYPE="ext4"
```

---

## 📌 Step 6.2: fstab में entry डालो

```bash
nano /etc/fstab
```

👉 Add this line:

```
UUID=abc123   /data   ext4   defaults,nofail   0   2
```

---

## 🔍 समझो ये line:

| Field       | Meaning                              |
| ----------- | ------------------------------------ |
| UUID=abc123 | disk पहचान                           |
| /data       | mount location                       |
| ext4        | filesystem                           |
| defaults    | normal settings                      |
| nofail      | boot fail नहीं होगा अगर disk missing |
| 0 2         | fsck settings                        |

---

## 📌 Step 6.3: Test करो

```bash
mount -a
```

👉 अगर error नहीं आया → सही है ✅

---

# 🔐 Permissions Setup

```bash
chown appuser:appgroup /data
chmod 775 /data
```

---

# 🧠 Real Production Tips

## 🔹 1. UUID ही use करो

👉 `/dev/sdb1` बदल सकता है reboot के बाद
👉 UUID stable होता है

---

## 🔹 2. nofail use करो

👉 अगर disk attach नहीं हुई → system boot रुकेगा नहीं

---

## 🔹 3. Backup folder यहीं रखो

```bash
/data/backups
```

---

# 🔥 Common Mistakes ❌

* `/dev/sdb1` directly fstab में use करना
* format भूल जाना
* wrong mount point
* permissions set नहीं करना

---

# 🚀 Full Flow Summary

```bash
lsblk
fdisk /dev/sdb
mkfs.ext4 /dev/sdb1
mkdir /data
mount /dev/sdb1 /data
blkid
nano /etc/fstab
mount -a
```

---

# 🎤 Interview Answer (Short)

> First, I identify the new disk using lsblk.
> Then I create a partition and format it with a filesystem.
> After that, I mount it to a directory.
> For persistence, I add an entry in /etc/fstab using UUID and test with mount -a.

---

# 🧠 Deep Insight

👉 Mount = “disk को system से जोड़ना”
👉 fstab = “system को याद दिलाना कि reboot के बाद भी जोड़ना है”

---

# ✅ Conclusion

👉 Temporary mount = testing
👉 Permanent mount = production

👉 Always:

* UUID use करो
* fstab सही लिखो
* mount -a से test करो

---
