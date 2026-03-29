# 🛠️ Nginx Advanced Debugging Guide: When Errors are "Silent"

Jab Nginx standard errors (404, 502, 500) nahi de raha, tab debugging "Deep Level" par karni padti hai.

---

### 1. Scenario: Status 200 OK but "Blank White Page"
Agar response code 200 hai par screen khali hai, toh iska matlab Nginx ne request process ki hai par content deliver nahi hua.

**Checklist:**
* **File Permissions:** Nginx user (`www-data`) ke paas files read karne ka haq nahi hai.
    * *Fix:* `sudo chown -R www-data:www-data /var/www/html`
* **Empty Upstream Response:** Agar Nginx proxy hai (Node/PHP ke liye), toh backend app ne "Empty Body" bheja hai.
* **Frontend Crash:** React/Next.js ka `index.html` load ho gaya par JavaScript runtime mein crash ho gayi (Check Browser Console).
* **FastCGI/Proxy Buffering:** Kabhi kabhi buffer size chota hone se content truncate (kat) jata hai.

---

### 2. Scenario: Connection Timeout (No Status Code)
Browser sirf "Rotating" dikha raha hai aur der baad 'Timed Out' bol deta hai.

**Checklist:**
* **Cloud Security Groups:** AWS/Azure mein Port 80/443 ka Inbound rule miss hai.
* **OS Firewall:** Server ke andar `ufw` ya `iptables` request block kar raha hai.
    * *Check:* `sudo ufw status`
* **Service Hung:** Nginx ki Master process chal rahi hai par Worker processes 'Zombie' ban chuki hain.
    * *Fix:* `sudo systemctl restart nginx`

---

### 3. Expert Debugging Commands (The Toolbox)

# A. Real-time Log Monitoring (Sabse Pehle Yeh Karein)
tail -f /var/log/nginx/access.log /var/log/nginx/error.log

# B. Configuration Syntax Validation
nginx -t

# C. Check if Nginx is actually listening on Ports
netstat -tuln | grep :80

# D. Test Local Response (Bypass Firewall/Browser)
curl -I localhost

# E. Resource Check (Disk Space & RAM)
df -h    # Agar disk 100% full hai, toh logs nahi likhenge aur request fail hogi.
top      # Check karein CPU usage 100% toh nahi.

---

### 💡 Interviewer Punchline:
"Sir, agar status codes silent hain, toh main **Physical Layer (Disk/RAM)**, **Network Layer (Firewall)**, aur **Permission Layer (User Ownership)** par focus karunga, kyunki Nginx logic tabhi fail hota hai jab use resources nahi milte."
