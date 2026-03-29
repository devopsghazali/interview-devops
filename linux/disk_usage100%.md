# 🐧 Linux Troubleshooting Guide: Disk 100% Full Case

Jab Linux server ki disk **100% full** ho jati hai, to services (Nginx, MySQL) crash hone lagti hain. Neeche ek **professional aur bareek tareeqa** diya gaya hai is masle ko diagnose aur fix karne ke liye.

---

## 🔍 Phase 1: Diagnosis (Beemari Kahan Hai?)

Sab se pehle hume **Bird's Eye View → Deep Dive** approach follow karni hoti hai.

### 📊 Disk Overview (Partition Level)
    df -h

### 📦 Folder-wise Size Check (/var)
    du -sh /var/* | sort -h

### 📁 Important Directories Samajh Lo
- `/var/log` → System ke **logs (CCTV records)**
- `/var/lib` → **Databases / Docker data (asli data)**
- `/var/cache` → **Temporary files (rough notebook)**

---

## 👻 Phase 2: The "Ghost Files" Problem (lsof)

Kabhi aisa hota hai:

- Aap file `rm` kar dete ho ❌  
- Lekin disk space free nahi hota ❗  

### 🤔 Kyun?
Kyuki file kisi **process ke paas open hoti hai**.

### 🔎 Check Ghost Files
    sudo lsof | grep deleted

### 🧠 Logic
Jab tak koi process file ko pakde hue hai:
- File disk se **actually delete nahi hoti**
- Space occupied rehta hai

---

## 🛠️ Phase 3: The Cure (Safai Ka Tareeqa)

### ✅ Method 1: Truncate (Professional Way)

Logs ko delete karne ke bajaye **zero size** kar dete hain:

    sudo truncate -s 0 /var/log/syslog

### 🎯 Fayda
- File ka naam same rehta hai
- Size instantly **0 KB** ho jata hai
- Service bina crash hue chalti rehti hai

---

### ⚠️ Method 2: Kill -9 (Last Resort)

Agar `lsof` mein ghost file mil jaye:

    sudo kill -9 <PID>

### 💣 Meaning
- `-9` = Force kill ("Shut up and die")
- Process turant band ho jata hai
- File release hoti hai → space free ho jata hai

---

## 📜 Master Cheat-Sheet

    # 🚀 Linux Disk Emergency Fix

    # Step 1: Check Global Space
    df -h

    # Step 2: Find the Heavy Folders
    sudo du -sh /var/* | sort -h

    # Step 3: Find Large Files (>500MB)
    sudo find / -type f -size +500M 2>/dev/null

    # Step 4: Check for Ghost Files
    sudo lsof | grep deleted

    # Step 5: Clean Logs safely
    sudo truncate -s 0 /var/log/*.log

    # Step 6: Kill process (if needed)
    sudo kill -9 <PID>

---

## 💡 Zaruri Tip

- Sabse pehle **/var/log** check karo  
- Agar Docker space le raha hai:

    docker system prune -a

- Regular log rotation enable karo:

    sudo logrotate -f /etc/logrotate.conf

---

## ✅ Summary

| Step | Action |
|------|-------|
| 1 | `df -h` se disk check |
| 2 | `du` se heavy folders dhundo |
| 3 | `lsof` se ghost files pakdo |
| 4 | `truncate` se logs safe clean karo |
| 5 | `kill -9` last option hai |

---

🔥 **Pro Tip:**  
Kabhi bhi blindly delete mat karo — pehle samjho **space kahan ja raha hai**, phir clean karo.
