╔══════════════════════════════════════════════════════════════════════════════╗
║ ⭐ Docker Permission Denied – Debugging Master Guide 🚀                      ║
╠══════════════════════════════════════════════════════════════════════════════╣

🎯 GOAL
Container chal raha hai lekin kaam nahi kar raha → kaise identify karein ki
yeh permission issue hai?

──────────────────────────────────────────────────────────────────────────────

🧠 CORE TRUTH

✔ Container running ≠ Application working properly  
❌ App silently fail kar sakti hai (especially write operations)

──────────────────────────────────────────────────────────────────────────────

🚨 STEP 1: Symptoms observe karo (FIRST SIGNAL)

Agar yeh cheezein ho rahi hain:

❌ File upload fail  
❌ Logs generate nahi ho rahe  
❌ Data save nahi ho raha  
❌ API 500 error de rahi hai  
❌ UI pe loading / incomplete response  

👉 Strong hint:
   “Application likh nahi pa rahi hai”

──────────────────────────────────────────────────────────────────────────────

🔥 STEP 2: Logs check karo (MOST IMPORTANT)

Command:

   docker logs <container_id>

Example:

   Error: EACCES: permission denied, open '/app/logs/app.log'

👉 Breakdown:
   EACCES = permission issue  
   Path mil gaya = /app/logs/app.log  

💥 Yeh direct proof hai

──────────────────────────────────────────────────────────────────────────────

⚠️ Hidden Cases

Kabhi direct “permission denied” nahi likha hota:

   Error: failed to write file

👉 Still same issue ho sakta hai

──────────────────────────────────────────────────────────────────────────────

🔍 STEP 3: Pattern identify karo

Agar:

✔ Read operations chal rahe hain  
❌ Write operations fail ho rahe hain  

👉 90% case:
   Permission issue 🔥

──────────────────────────────────────────────────────────────────────────────

🧠 STEP 4: Logical deduction (Interview level)

Situation:

✔ App chal rahi hai  
✔ API hit ho rahi hai  
❌ Data persist nahi ho raha  

👉 Possible causes:
   - DB issue
   - Network issue
   - Permission issue

👉 Narrow down:

✔ DB connected → OK  
✔ API working → OK  

👉 Final suspicion:
   FILE PERMISSION ISSUE ✅

──────────────────────────────────────────────────────────────────────────────

🧪 STEP 5: Manual confirmation (FINAL PROOF)

Container ke andar jao:

   docker exec -it <container_id> bash

Phir test karo:

   cd /app
   touch test.txt

👉 Result:

❌ Permission denied → CONFIRMED  
✔ File create ho gaya → issue nahi hai

──────────────────────────────────────────────────────────────────────────────

🔍 STEP 6: File/FOLDER permissions check

   ls -l
   ls -ld /app/logs

Example:

   -rw-r--r-- 1 root root app.log

👉 Samjho:

   Owner = root  
   Current user ≠ root  

❌ Write allowed nahi

──────────────────────────────────────────────────────────────────────────────

🔍 STEP 7: Current user identify karo

   whoami

Example:

   node

👉 Mismatch:

   File owner = root  
   Process user = node  

💥 Root cause mil gaya

──────────────────────────────────────────────────────────────────────────────

⚡ COMPLETE DEBUG FLOW

1️⃣ App ka behavior observe karo  
2️⃣ docker logs check karo  
3️⃣ Error/path identify karo  
4️⃣ Container ke andar jao  
5️⃣ whoami → current user  
6️⃣ ls -l → file owner check  
7️⃣ Manual write test (touch/echo)  
8️⃣ Confirm → permission issue  

──────────────────────────────────────────────────────────────────────────────

💡 INTUITION BUILD KARO

👉 Jab bhi:

- “save / upload / write” fail ho  
- lekin “read” chal raha ho  

👉 Turant socho:

   🔥 “Yeh permission issue ho sakta hai”

──────────────────────────────────────────────────────────────────────────────

🎯 ONE-LINE INTERVIEW ANSWER

“Main pehle application behavior observe karta hoon, phir logs check karta
hoon jahan EACCES ya write errors milte hain. Uske baad container ke andar
jaake user aur file permissions compare karke manual write test se confirm
karta hoon.”

──────────────────────────────────────────────────────────────────────────────

💡 MASTER LINE

👉 “App chal rahi ho lekin data write na ho raha ho → permission issue check karo”

╚══════════════════════════════════════════════════════════════════════════════╝
