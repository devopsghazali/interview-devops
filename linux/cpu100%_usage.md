# 🔥 CPU 99% Troubleshooting Guide (Linux) – Deep Dive

High CPU usage ek serious issue hai jo server ko slow, unresponsive ya crash tak kar sakta hai. Ye guide aapko **step-by-step professional approach** deta hai: Identify → Investigate → Act → Fix Root Cause.

---

## 🔍 Phase 1: Identify (Kaun Shor Macha Raha Hai?)

Sabse pehle hume ye pata lagana hai ke **CPU kaun kha raha hai**.

### 📊 Real-time Monitoring
    top

### 🎨 Better View (Recommended)
    htop

### 📋 Top CPU Processes List
    ps aux --sort=-%cpu | head -n 10

---

### 🧠 Bareek Insight
- Kabhi ek hi process 99% leta hai  
- Kabhi 100 chhote processes milkar CPU ko overload kar dete hain  
- Isliye sirf ek PID par focus mat karo — pattern dekho  

---

## 🔎 Phase 2: Drill Down (Andar Kya Chal Raha Hai?)

Sirf PID milna kaafi nahi — hume samajhna hai **process kar kya raha hai**.

### 🎯 Single Process Monitor
    top -p <PID>

👉 Sirf us process ki live activity dikhegi

---

### 🧬 Deep Debug (Expert Level – X-Ray)
    strace -p <PID>

### 🧠 Logic
- Ye system calls show karta hai (kernel se kya maang raha hai)
- Agar same cheez repeat ho rahi hai:
  👉 Infinite loop / stuck process

---

### 🚨 Red Flags
- Same function baar-baar → loop
- Disk read/write repeat → I/O issue
- Network calls repeat → API issue

---

## ⚡ Phase 3: Kill Strategy (Tameez vs Zardasti)

### 😇 Method 1: Graceful Kill (Recommended First)
    kill -15 <PID>

### 🧠 Logic
- Process ko signal milta hai: "Shanti se band ho jao"
- Wo:
  - Files save karta hai  
  - Connections close karta hai  
  - Proper exit karta hai  

✔ Safe  
✔ Clean  

---

### 💣 Method 2: Force Kill (Last Resort)
    kill -9 <PID>

### 🧠 Logic
- Direct terminate (no cleanup)
- "Shut up and die"

⚠️ Risk:
- Data loss  
- Corruption  

👉 Sirf tab use karo jab:
- Process hang ho  
- -15 kaam na kare  

---

## 🧠 Phase 4: Root Cause Analysis (Asli Wajah)

CPU kab high hota hai? Jab system ko zyada "sochna" padta hai.

---

### 🔁 1. Infinite Loop
Example:
    while(true)

### 🔍 Symptom
- Constant high CPU
- Same pattern repeat

---

### 🧠 2. Memory Leak
### 🔍 Symptom
- RAM full → swap use
- CPU spike (swap handling)

---

### 🗄️ 3. Bad Database Query
### 🔍 Symptom
- Slow queries
- High CPU on DB server

### ❗ Cause
- Missing index
- Full table scan

---

### 🌐 4. High Traffic / DDoS
### 🔍 Symptom
- Sudden spike
- Multiple connections

---

## 🛠️ Bonus: Control CPU (Nice / Renice)

### 😇 Process ko "Polite" banao
    nice -n 10 command

### 🔧 Existing Process
    renice 10 -p <PID>

### 🧠 Rule
- Nice ↑ → Priority ↓ → CPU kam milega  
- Nice ↓ → Priority ↑ → CPU zyada milega  

---

## 📜 Master Cheat-Sheet

    # 🔍 Identify
    htop
    ps aux --sort=-%cpu | head -n 10

    # 🔎 Investigate
    top -p <PID>
    strace -p <PID>

    # ⚡ Action
    kill -15 <PID>
    kill -9 <PID>

    # 🎛 Control
    renice 10 -p <PID>

---

## 💡 Bareek Expert Tip

Agar aap `top` me ye dekhein:

    wa (wait) high hai

👉 Matlab:
- CPU free hai  
- Lekin disk slow hai  
- CPU wait kar raha hai  

🔥 Real problem = **Disk I/O bottleneck**, not CPU

---

## 🎯 Final Mindset

> "CPU troubleshoot karna sirf process kill karna nahi hai —  
> asli kaam hai samajhna ke system itna soch kyun raha hai"

---

## ✅ Quick Checklist

- [ ] Top processes identify kiye?
- [ ] Single PID analyze kiya?
- [ ] strace se behavior dekha?
- [ ] Graceful kill try kiya?
- [ ] Root cause identify kiya?

---

🚀 **Golden Rule:**  
Pehle samjho → phir fix karo → warna problem wapas aayegi
