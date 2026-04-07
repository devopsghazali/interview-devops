╔══════════════════════════════════════════════════════════════════════════════╗
║ 🌐 Master Doc: Solving Docker Port Mapping Issues 🚀                         ║
╠══════════════════════════════════════════════════════════════════════════════╣

📌 Quick Overview
Docker containers isolated network namespaces mein chalte hain. Browser se 
access karne ke liye humein ek "Tunnel" ya "Bridge" banana padta hai jise 
hum Port Forwarding kehte hain.

🛠️ The Core Logic: Host-to-Container Bridge
Jab aap -p flag use karte hain, toh aap Docker engine ko batate hain ki 
laptop ke request ko container ke andar kaise route karna hai.

👉 The Golden Rule:
   -p <Host_Port>:<Container_Port>

🔍 Step-by-Step Troubleshooting Flow

1️⃣ Identify the Internal Port
Pehle ye confirm karein ki application container ke andar kis port par 
'listen' kar rahi hai.

✔ Dockerfile check karein:
   EXPOSE 3000

✔ Ya CMD / app config check karein

✔ Running Container check karein:
   docker ps

⚠️ Agar "PORTS" section khali hai ya sirf 3000/tcp dikha raha hai bina 
   arrows (->) ke, toh mapping missing hai.

──────────────────────────────────────────────────────────────────────────────

2️⃣ The "0.0.0.0" Binding (CRITICAL)

Aapki application ka code localhost (127.0.0.1) par nahi, balki 
0.0.0.0 par bind hona chahiye.

❓ Kyun?
- 127.0.0.1 = sirf container ke andar accessible
- 0.0.0.0 = external requests (Docker bridge se) allow karta hai

──────────────────────────────────────────────────────────────────────────────

3️⃣ Correct Execution Command

Agar app container ke andar port 3000 par chal raha hai aur aap usse 
browser mein localhost:8080 par dekhna chahte hain:

   docker run -d \
     --name my-app \
     -p 8080:3000 \
     myapp-image

──────────────────────────────────────────────────────────────────────────────

📋 Common Scenarios & Fixes

🔴 Connection Refused
Reason: Container exit ho gaya ya mapping missing hai
Fix: docker ps check karein + -p flag add karein

🔴 Site Can't Be Reached
Reason: Host port already busy hai (e.g. 80)
Fix: Port change karein → -p 9000:80

🔴 Empty Response
Reason: App 127.0.0.1 par bind hai
Fix: Host ko 0.0.0.0 karein

🔴 Cloud Access Issue
Reason: Security Groups block kar rahe hain
Fix: Cloud dashboard (AWS/GCP) mein inbound port allow karein

──────────────────────────────────────────────────────────────────────────────

⚡ Interactive Q&A for Debugging

Q: Laptop ka port free hai ya nahi kaise check karein?

✔ Windows:
   netstat -ano | findstr :8080

✔ Linux/Mac:
   lsof -i :8080

--------------------------------------------------

Q: Kya multiple ports map kar sakte hain?

✔ Haan:
   docker run -p 80:80 -p 443:443 my-proxy

--------------------------------------------------

Q: Docker Compose mein kaise likhein?

services:
  web:
    image: myapp
    ports:
      - "8080:3000"   # Host:Container

──────────────────────────────────────────────────────────────────────────────

💡 Gehra Flow Tip (DevSecOps Perspective)

Jab aap Appwrite ya React Native backend deploy karte hain:

✔ Backend aur frontend alag containers mein hote hain
✔ Unhe same Docker Network mein rakhein
✔ Internal communication ke liye ports expose karne ki zarurat nahi hoti

👉 Benefit:
- Secure communication
- Better isolation
- Cleaner architecture

╚══════════════════════════════════════════════════════════════════════════════╝
