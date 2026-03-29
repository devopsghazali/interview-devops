```
╔══════════════════════════════════════════════════════════════════════════════╗
║              🚨 SSH NOT CONNECTING - COMPLETE DEBUG GUIDE 🚨                ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🧠 GOAL:                                                                     ║
║ SSH connection kyun fail ho rahi hai → identify + fix                        ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🔍 STEP 1: ERROR MESSAGE SAMJHO (MOST IMPORTANT)                             ║
║                                                                              ║
║ ssh user@ip                                                                  ║
║                                                                              ║
║ ❗ Possible errors:                                                           ║
║ ➤ Connection timed out   → network/firewall issue                            ║
║ ➤ Connection refused     → SSH service down                                  ║
║ ➤ Permission denied      → auth/key issue                                    ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🌐 STEP 2: SERVER REACHABLE HAI YA NAHI                                      ║
║                                                                              ║
║ ping <server-ip>                                                             ║
║                                                                              ║
║ ❌ Fail → server down / network issue                                         ║
║ ✔ Pass → next step                                                           ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🔌 STEP 3: PORT 22 OPEN HAI YA NAHI                                          ║
║                                                                              ║
║ nc -zv <server-ip> 22                                                        ║
║                                                                              ║
║ ❌ Closed → firewall / security group issue                                  ║
║ ✔ Open → next step                                                           ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ ☁️ STEP 4: CLOUD CHECKS (AWS/AZURE/GCP)                                       ║
║                                                                              ║
║ ✔ Instance running hai                                                       ║
║ ✔ Public IP correct hai                                                      ║
║ ✔ Security Group: Port 22 ALLOW (your IP)                                    ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🖥 STEP 5: SSH SERVICE STATUS (SERVER SIDE)                                   ║
║                                                                              ║
║ systemctl status ssh                                                         ║
║                                                                              ║
║ ❌ Inactive →                                                                ║
║ systemctl start ssh                                                          ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🔥 STEP 6: FIREWALL CHECK                                                    ║
║                                                                              ║
║ ufw status                                                                   ║
║                                                                              ║
║ ❌ Port 22 not allowed →                                                     ║
║ ufw allow 22                                                                 ║
║ ufw enable                                                                   ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 👤 STEP 7: AUTHENTICATION CHECK                                              ║
║                                                                              ║
║ ssh -i key.pem user@ip                                                       ║
║                                                                              ║
║ ✔ Correct key use karo                                                       ║
║ ✔ Permission fix:                                                            ║
║ chmod 400 key.pem                                                            ║
║                                                                              ║
║ ❗ Error: Permission denied (publickey)                                       ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ ⚠️ STEP 8: DISK FULL CHECK (HIDDEN ISSUE)                                    ║
║                                                                              ║
║ df -h                                                                        ║
║                                                                              ║
║ ❌ /var full → SSH fail ho sakta hai                                          ║
║                                                                              ║
║ ✔ Fix:                                                                       ║
║ journalctl --vacuum-time=1d                                                   ║
║ apt-get clean                                                                ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 📜 STEP 9: LOGS CHECK (ROOT CAUSE)                                           ║
║                                                                              ║
║ journalctl -u ssh                                                            ║
║                                                                              ║
║ ya                                                                            ║
║                                                                              ║
║ cat /var/log/auth.log                                                        ║
║                                                                              ║
║ 👉 Yahan exact error milega                                                  ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 🎯 FINAL FLOW (REMEMBER THIS)                                                ║
║                                                                              ║
║ 1. Error message                                                             ║
║ 2. Network (ping)                                                            ║
║ 3. Port check                                                                ║
║ 4. Cloud config                                                              ║
║ 5. SSH service                                                               ║
║ 6. Firewall                                                                  ║
║ 7. Auth/key                                                                  ║
║ 8. Disk space                                                                ║
║ 9. Logs                                                                      ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║ 💡 GOLDEN RULE                                                               ║
║                                                                              ║
║ "First stabilize, then investigate deeply"                                  ║
║                                                                              ║
║ 🔥 80% issues = security group / key / ssh service                           ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```
