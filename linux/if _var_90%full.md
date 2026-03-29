```
╔══════════════════════════════════════════════════════════════════╗
║           🚀 LINUX LOGGING & JOURNALCTL MASTER NOTES            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🧠 1. LOGGING KYA HOTA HAI?                                      ║
║                                                                  ║
║ Logs = system ki history 📜                                      ║
║                                                                  ║
║ ➤ Kaun login hua                                                 ║
║ ➤ Kaunsi service start/stop hui                                  ║
║ ➤ Error kaha aaya                                                ║
║                                                                  ║
║ 💡 DevOps mein logs = debugging ka sabse bada weapon             ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ ⚙️ 2. SYSTEMD & JOURNAL                                          ║
║                                                                  ║
║ Modern Linux → systemd use karta hai                             ║
║                                                                  ║
║ ➤ systemd logs ko ek jagah store karta hai → journal             ║
║                                                                  ║
║ 📦 Location:                                                     ║
║ /var/log/journal                                                 ║
║                                                                  ║
║ 🎯 Access tool = journalctl                                      ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🔍 3. JOURNALCTL BASIC COMMANDS                                  ║
║                                                                  ║
║ ➤ Saare logs:                                                    ║
║   journalctl                                                     ║
║                                                                  ║
║ ➤ Live logs:                                                     ║
║   journalctl -f                                                  ║
║                                                                  ║
║ ➤ Last 50 logs:                                                  ║
║   journalctl -n 50                                               ║
║                                                                  ║
║ ➤ Specific service:                                              ║
║   journalctl -u nginx                                            ║
║                                                                  ║
║ ➤ Current boot:                                                  ║
║   journalctl -b                                                  ║
║                                                                  ║
║ ➤ Previous boot:                                                 ║
║   journalctl -b -1                                               ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ ⚡ 4. NGINX LOGGING FLOW                                         ║
║                                                                  ║
║ 📁 Application logs:                                             ║
║ /var/log/nginx/access.log                                        ║
║ /var/log/nginx/error.log                                         ║
║                                                                  ║
║ 📡 System logs:                                                  ║
║ journalctl -u nginx                                              ║
║                                                                  ║
║ 💡 Difference:                                                   ║
║ Access log → user requests                                       ║
║ Journal → service events                                         ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🚨 5. DISK FULL SCENARIO (/var 90%+)                             ║
║                                                                  ║
║ 🔍 Step 1: Check disk                                            ║
║ df -h /var                                                       ║
║                                                                  ║
║ 🔍 Step 2: Find culprit                                          ║
║ du -sh /var/* | sort -h                                          ║
║                                                                  ║
║ ⚡ Step 3: Emergency cleanup                                      ║
║                                                                  ║
║ ➤ Old logs delete:                                               ║
║ journalctl --vacuum-time=1d                                      ║
║                                                                  ║
║ ➤ Size limit:                                                    ║
║ journalctl --vacuum-size=100M                                    ║
║                                                                  ║
║ ➤ Big files:                                                     ║
║ find /var -type f -size +100M                                    ║
║                                                                  ║
║ ➤ Truncate logs:                                                 ║
║ truncate -s 0 /var/log/syslog                                    ║
║                                                                  ║
║ ➤ Cache clean:                                                   ║
║ apt-get clean                                                    ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🧹 6. VACUUM-TIME EXPLAINED                                      ║
║                                                                  ║
║ journalctl --vacuum-time=1d                                      ║
║                                                                  ║
║ 👉 Meaning:                                                      ║
║ 1 din se purane logs delete                                      ║
║                                                                  ║
║ Examples:                                                        ║
║ ➤ 7d = 7 din purane delete                                       ║
║ ➤ 1h = 1 ghante purane delete                                    ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🔥 7. HIDDEN ISSUE (DELETED BUT STILL USED)                      ║
║                                                                  ║
║ lsof | grep deleted                                              ║
║                                                                  ║
║ 👉 File delete ho gayi lekin process use kar raha hai            ║
║                                                                  ║
║ ✔ Fix:                                                           ║
║ systemctl restart <service>                                      ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🛡 8. PERMANENT FIX                                               ║
║                                                                  ║
║ ✔ Logrotate enable karo                                          ║
║ ✔ Monitoring alerts lagao                                        ║
║ ✔ Separate partition (/var/log)                                  ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ 🎯 FINAL INTERVIEW SUMMARY                                       ║
║                                                                  ║
║ ➤ Identify disk issue (df, du)                                   ║
║ ➤ Free space quickly (vacuum, truncate)                          ║
║ ➤ Check hidden files (lsof)                                      ║
║ ➤ Fix root cause (logrotate, monitoring)                         ║
║
```
