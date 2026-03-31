# 🚨 Build Local Pass, CI (Jenkins/GitHub Actions) Fail — Complete Troubleshooting Guide

---

# 🧠 Problem Statement

```
Local:
✅ Build pass

CI (Jenkins / GitHub Actions):
❌ Build fail
```

👉 Root cause:
**Environment mismatch (local vs CI)**

---

# 🎯 Goal

👉 Jab CI fail ho, to systematically troubleshoot karo  
👉 Guess nahi — **structured debugging approach**

---

# 🚀 STEP-BY-STEP TROUBLESHOOTING

---

# 🔍 1. Logs Check Karna (SABSE PEHLA STEP)

👉 Always start with:

```
Jenkins → Console Output
GitHub Actions → Logs
```

---

## ✅ Kya dekhna hai?

- exact error message
- kis step pe fail hua
- dependency / permission / env issue

---

## 💥 Example

```
Permission denied
```

```
Could not resolve dependency
```

```
command not found
```

---

# ⚙️ 2. Same Command CI Server pe Run Karna

👉 CI jo command run kar raha hai:

```
mvn clean install
npm install
docker build
```

👉 usko manually CI machine pe run karo

---

## 🎯 Purpose

👉 confirm karo:

```
issue code ka hai ya environment ka
```

---

# 🌍 3. Environment Variables Check Karna

---

## ❓ Problem

👉 `.env` file Git me push nahi hoti

👉 CI me variables missing ho sakte hain

---

## 🔍 Check

```
printenv
echo $DB_HOST
```

---

## ❌ Error Example

```
DB connection failed
API_KEY undefined
```

---

## ✅ Fix

👉 Jenkins:

- Manage Jenkins → Credentials / Env variables

👉 GitHub Actions:

- Settings → Secrets

---

# 🔐 4. Credentials Check Karna

---

## ❓ Problem

👉 Secrets missing ya galat ho sakte hain

---

## 💥 Example

```
docker login failed
aws authentication failed
```

---

## 🔍 Check

- username/password
- tokens
- API keys

---

## ✅ Fix

👉 Correct secrets configure karo:

- Jenkins → Credentials
- GitHub → Secrets

---

# 🔑 5. File Permission Check Karna

---

## ❓ Problem

👉 Jenkins user ≠ Local user

---

## 💥 Example

```
Permission denied
./script.sh not executable
```

---

## 🔍 Check

```
ls -l
whoami
```

---

## ✅ Fix

```
chmod +x script.sh
chmod -R 755 .
```

---

## 🧠 Note

👉 Git sirf executable flag store karta hai  
👉 baaki permissions CI environment decide karta hai

---

# 📦 6. Dependency Issue Check Karna

---

## ❓ Problem

👉 Local me cached dependencies hoti hain  
👉 CI me fresh environment hota hai

---

## 💥 Example

```
Could not resolve dependency
module not found
```

---

## 🔍 Check

- internet access
- repo access
- version mismatch

---

## ✅ Fix

```
mvn dependency:go-offline
npm install
pip install -r requirements.txt
```

---

## 🧠 Note

👉 Maven cache:

```
~/.m2/repository
```

👉 CI me empty hota hai

---

# 🐳 7. Tool / Runtime Missing Check

---

## ❓ Problem

👉 CI machine pe required tools installed nahi hote

---

## 💥 Example

```
mvn: command not found
node: not found
docker: not found
```

---

## ✅ Fix

👉 Install tools in pipeline:

```
sudo apt install maven
sudo apt install nodejs
```

---

# 📁 8. Path / File Missing Issue

---

## ❓ Problem

👉 file exist nahi karti CI me

---

## 💥 Example

```
file not found
No such file or directory
```

---

## 🔍 Check

```
pwd
ls -la
```

---

## ✅ Fix

👉 correct path use karo

---

# 🔄 9. Clean Build Try Karna

---

## ❓ Problem

👉 cache ya stale files issue create karte hain

---

## ✅ Fix

```
mvn clean install
rm -rf node_modules
npm install
```

---

# 🧠 10. Local vs CI Comparison

---

| Feature | Local | CI |
|--------|------|----|
| Environment | Stable | Fresh |
| Cache | Available | Missing |
| Permissions | Full | Restricted |
| Secrets | Available | Manual setup |

---

# 🔥 FINAL CHECKLIST

---

👉 Jab CI fail ho:

```
1. Logs dekho
2. Same command manually run karo
3. Env variables check karo
4. Credentials verify karo
5. File permissions check karo
6. Dependencies check karo
7. Tools installed hain ya nahi dekho
8. Path verify karo
9. Clean build try karo
```

---

# 🎯 FINAL UNDERSTANDING

👉

```
Local works because everything is already configured  
CI fails because it runs in a clean, isolated environment
```

---

# 🔥 INTERVIEW GOLD LINE

👉

> "When a build passes locally but fails in CI, I follow a structured approach:  
> check logs, verify environment variables and credentials, ensure correct file permissions, and confirm dependencies and tools are properly installed in the CI environment."

---

# 🚀 BONUS TIP

👉 Always write pipeline like:

```
idempotent + reproducible
```

👉 Matlab:

```
same result every time (local = CI)
```

---
