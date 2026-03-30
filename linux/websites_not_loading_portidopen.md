# 🌐 Nginx Troubleshooting Guide (Deep Dive for Beginners → DevOps)

## 📌 Scenario

Aapki **website load nahi ho rahi**, lekin **port open hai**.

👉 Iska matlab: network level pe request aa rahi hai, lekin application (nginx ya backend) me issue ho sakta hai.

Is guide me hum **step-by-step systematic debugging approach** seekhenge.

---

# 🧠 Troubleshooting Mindset

👉 Rule:

> "Top se bottom nahi, layer-by-layer check karo"

Flow:

1. Service running?
2. Port listening?
3. Logs kya keh rahe hain?
4. Files sahi jagah hain?
5. Config sahi hai?
6. Backend issue to nahi?

---

# ⚙️ 1. Check: Nginx Service Running Hai Ya Nahi

```bash
systemctl status nginx
```

## 🧠 Kya dekhna hai?

* Active: running ✅
* inactive / failed ❌

---

## 🔥 Quick check:

```bash
systemctl is-active nginx
```

👉 Output:

* `active` → service chal rahi hai
* `inactive` → service band hai

---

## ❌ Agar nginx down hai:

```bash
systemctl start nginx
systemctl restart nginx
```

👉 Agar start nahi ho raha → logs check karo (next step)

---

# 🔌 2. Check: Port Listening Hai Ya Nahi

```bash
ss -tulnp | grep :80
```

## 🧠 Explanation:

| Part     | Meaning                     |
| -------- | --------------------------- |
| ss       | socket status               |
| -tulnp   | TCP/UDP listening processes |
| grep :80 | port 80 filter              |

---

## 🔍 Output Example:

```
LISTEN 0 128 0.0.0.0:80 ... nginx
```

👉 Matlab nginx port 80 pe listen kar raha hai

---

## ❌ Agar port open hai but nginx nahi:

👉 Koi aur service port use kar rahi ho sakti hai

---

# 📜 3. Check Logs (MOST IMPORTANT STEP)

## 🔹 Error log:

```bash
cat /var/log/nginx/error.log
```

---

## 🔹 Live monitoring:

```bash
tail -f /var/log/nginx/error.log
```

---

## 🧠 Common Errors:

| Error              | Meaning           |
| ------------------ | ----------------- |
| permission denied  | file access issue |
| no such file       | file missing      |
| connection refused | backend down      |

---

# 📁 4. Check HTML / Website Files

## 🔹 Default path:

```bash
/var/www/html
```

---

## 🔹 Check files:

```bash
ls -l /var/www/html
```

---

## 🧠 Kya check kare?

* index.html present hai?
* permissions sahi hain?
* correct folder configured hai?

---

# 🔐 5. Check Permissions

```bash
ls -l /var/www/html
```

👉 Example:

```
-rw-r--r-- 1 root root index.html
```

---

## ❌ Issue:

* nginx user ko read access nahi

---

## ✅ Fix:

```bash
chown -R www-data:www-data /var/www/html
chmod -R 755 /var/www/html
```

---

# ⚙️ 6. Check Nginx Config File

## 🔹 Main config:

```bash
/etc/nginx/nginx.conf
```

## 🔹 Site config:

```bash
/etc/nginx/sites-available/
```

---

## 🔍 Test config:

```bash
nginx -t
```

👉 Output:

```
syntax is ok
```

---

## ❌ Agar error aaye:

👉 line number ke saath error batayega

---

# 🔁 7. Reload Config

```bash
systemctl reload nginx
```

👉 Restart nahi, sirf config reload

---

# 🔗 8. Backend Check (Advanced)

Agar nginx reverse proxy hai:

```bash
curl http://127.0.0.1:3000
```

👉 Backend service chal rahi hai ya nahi?

---

# 🌍 9. Browser / Curl Test

```bash
curl -I http://localhost
```

👉 HTTP response check karega

---

# 🔥 10. Full Debug Flow (Real Interview Answer)

👉 Agar interviewer pooche:

**"Website load nahi ho rahi, kya karoge?"**

### ✅ Perfect Answer:

1. Check karunga nginx running hai ya nahi
2. Port listening check karunga
3. Logs check karunga (/var/log/nginx/error.log)
4. Website files verify karunga (/var/www/html)
5. Permissions check karunga
6. Nginx config validate karunga (nginx -t)
7. Backend service check karunga (agar proxy hai)

---

# 🧠 Key Concepts Summary

| Step      | Purpose            |
| --------- | ------------------ |
| systemctl | service status     |
| ss        | port check         |
| logs      | error detection    |
| files     | content validation |
| config    | syntax + routing   |
| backend   | dependency check   |

---

# 🚀 Final Summary

👉 Website issue = sirf nginx nahi, multiple layers ka problem ho sakta hai
👉 Logs sabse powerful debugging tool hain
👉 Step-by-step approach hi real DevOps skill hai

---

# 🔥 Next Level (Optional)

* Load balancer debugging
* 502 / 504 error deep dive
* SSL issues troubleshooting
* High traffic failure analysis
