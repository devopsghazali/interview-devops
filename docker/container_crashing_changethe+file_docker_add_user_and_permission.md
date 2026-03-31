# 🐳 Docker Container Crash Due to Permission Issues (Detailed Guide)

## 🎯 Goal

Is document ka purpose hai samajhna:

* Container **kyun crash ho raha tha**
* Problem ka **root cause kya tha**
* Humne ise **kaise fix kiya (step-by-step)**

---

# 🧠 Problem Statement

Humne Docker container mein:

* Ek **non-root user (`appuser`)** banaya
* Aur container ko us user se run kar diya

Lekin container run karte hi error aaya:

```bash
Permission denied
```

---

# 🔍 Root Cause (Asli Problem)

👉 Docker container by default **root user** se build hota hai
👉 Lekin humne runtime pe user change kar diya:

```dockerfile
USER appuser
```

### ❗ Issue kya hua?

* Files aur directories ka owner = **root**
* Container run ho raha hai = **appuser**

👉 Matlab:

> `appuser` ke paas required permissions nahi thi

---

# 💥 Failure Scenario (Broken Example)

```dockerfile
FROM ubuntu:22.04

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
RUN touch file.txt

USER appuser

CMD ["cat", "/app/file.txt"]
```

---

## ❌ Result:

```bash
cat: /app/file.txt: Permission denied
```

---

# 🧠 Why It Failed?

| Component | Owner | Running User | Problem     |
| --------- | ----- | ------------ | ----------- |
| file.txt  | root  | appuser      | ❌ No access |

👉 appuser ke paas:

* read permission ❌
* write permission ❌

---

# ✅ Solution (Fix)

## 🔧 Step 1: Ownership change karo

```dockerfile
RUN chown -R appuser:appgroup /app
```

---

## ✔️ Fixed Dockerfile

```dockerfile
FROM ubuntu:22.04

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
RUN touch file.txt

# 🔥 Fix: ownership change
RUN chown -R appuser:appgroup /app

USER appuser

CMD ["cat", "/app/file.txt"]
```

---

## ✅ Result:

```bash
(file successfully read)
```

👉 Container ab crash nahi karega 🎯

---

# ⚡ Better Approach (Optimized)

👉 Direct copy ke time ownership set karo:

```dockerfile
COPY --chown=appuser:appgroup . .
```

---

# 🧠 Another Common Crash Case

## ❌ Low Port Binding Issue

```js
app.listen(80)
```

👉 Problem:

* Ports `<1024` → root required
* Non-root user fail karega

---

## ✅ Fix:

```js
app.listen(3000)
```

---

# 🧠 Golden Rule

> **Jis user se container run ho raha hai, usko required files aur directories ka access hona hi chahiye**

---

# 🔥 Best Practice Pattern

```dockerfile
FROM ubuntu:22.04

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
COPY . .

RUN chown -R appuser:appgroup /app

USER appuser

CMD ["your-app"]
```

---

# ⚠️ Common Mistakes

### ❌ Mistake 1

USER set kiya but permissions nahi di

---

### ❌ Mistake 2

Root-owned files pe non-root user run kar diya

---

### ❌ Mistake 3

Low ports (80, 443) use kiye without root

---

# 🧠 Final Understanding

## 💥 Container crash kab hota hai?

* Jab user ko access nahi milta
* Ya restricted operation karta hai

---

## ✅ Container stable kab hota hai?

* Proper ownership ho
* Correct permissions ho
* Suitable ports use ho

---

# 🏁 Final Summary

| Concept       | Explanation                    |
| ------------- | ------------------------------ |
| Root user     | Default Docker behavior        |
| Non-root user | Secure but needs permissions   |
| chown         | Ownership fix karta hai        |
| USER          | Runtime identity set karta hai |

---

# 🔥 One-Line Conclusion

> **Container crash ka main reason tha permission issue — aur humne use solve kiya ownership (`chown`) aur correct configuration se**

---
