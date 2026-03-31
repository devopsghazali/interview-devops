# 🐳 Docker Container Exit / Crash Troubleshooting (5 Steps Guide)

## 🧠 Problem
Container start hota hai aur turant exit ho jata hai:
- Exited (0) / Exited (1) / Exited (127)
- Container run nahi kar raha properly

---

# 🚀 Step 1: Container Status Check
docker ps -a

🔍 What to look:
- Exited (0) → normal exit
- Exited (1) → error/crash
- Exited (127) → command not found

---

# 📜 Step 2: Logs Check (MOST IMPORTANT)
docker logs <container_id>

🔍 What you get:
- application crash reason
- missing dependency
- config error
- runtime error

👉 Without logs = blind debugging ❌

---

# 🔍 Step 3: Inspect Container Config
docker inspect <container_id>

Check:
- Cmd
- Entrypoint
- Env variables

👉 Ensure correct startup command is set

---

# ⚙️ Step 4: Interactive Debug Mode
docker run -it --entrypoint /bin/bash <image_name>

Then inside container:
ls
ps aux
node app.js / python app.py

👉 Direct error yahin dikhega

---

# 🧠 Step 5: Fix Common Root Causes

## ❌ Case A: Main process exit
CMD ["echo", "Hello"]

Fix:
CMD ["nginx", "-g", "daemon off;"]

---

## ❌ Case B: Background process (&)
node app.js &

Fix:
node app.js

---

## ❌ Case C: Missing command
docker run ubuntu invalidcmd

Fix:
- correct command use karo
- proper image use karo

---

## ❌ Case D: Permission issue
permission denied /var/run/docker.sock

Fix:
sudo usermod -aG docker $USER
newgrp docker

---

# 🎯 Golden Rule
Docker container tab tak chalta hai jab tak MAIN PROCESS (PID 1) alive ho

---

# 🚀 Quick Debug Flow (Short Version)
1. docker ps -a
2. docker logs <container_id>
3. docker inspect <container_id>
4. docker run -it --entrypoint /bin/bash <image_name>
5. Fix CMD / ENTRYPOINT / app error

---

# 💡 Summary
Most container exits happen due to:
- wrong CMD
- app crash
- background process
- missing dependency
- permission issue

---
