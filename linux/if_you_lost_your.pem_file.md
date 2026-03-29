# 🛡️ EC2 SSH Key Recovery Guide (The Volume Swap Method)

Agar aapne apni .pem file kho di hai, toh is single-box guide ko follow karein. 
Ismein hum "Instance A" (Locked) ka dil nikaal kar "Instance B" (Healthy) mein lagayenge.

---

### 1. Preparation (AWS Console)
1. Stop Instance A (jiski key kho gayi hai).
2. Detach Root Volume: 'Volumes' tab mein jayein -> Instance A ka volume select karein -> 'Detach'.
3. Launch Instance B: Ek naya temporary instance banayein (Same Availability Zone mein).
4. Attach Volume to B: Instance A wala volume Instance B mein attach karein (Device: /dev/sdf).

---

### 2. Recovery Commands (Inside Instance B Terminal)

# Step A: Root access lein aur volume check karein
sudo -i
lsblk    # Yahan check karein aapka volume 'xvdf' ya 'nvme1n1' naam se dikhega

# Step B: Mount point banayein aur mount karein
mkdir /mnt/recovery
mount /dev/xvdf1 /mnt/recovery    # Agar error aaye toh 'lsblk' se partition name confirm karein

# Step C: Key Overwrite (The Magic Step)
# Instance B ki working key ko Instance A ke volume mein copy karna
cp ~/.ssh/authorized_keys /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Step D: Permissions aur Ownership Fix (Crucial!)
# 'ec2-user' ke liye UID 1000 hoti hai. Agar Ubuntu hai toh 'ubuntu' path use karein.
chown 1000:1000 /mnt/recovery/home/ec2-user/.ssh/authorized_keys
chmod 600 /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Step E: Cleanup
umount /mnt/recovery
exit

---

### 3. Restore (AWS Console)
1. Detach Volume: Instance B se volume ko detach karein.
2. Re-attach to Instance A: Volume ko wapas Instance A par lagayein.
   ⚠️ IMPORTANT: Device name hamesha '/dev/xvda' ya '/dev/sda1' hi rakhein.
3. Start Instance A: Ab aap Instance B wali .pem file se login kar payenge!

---

### 4. Verification Command (From your PC)
ssh -i "nayi-key.pem" ec2-user@<Instance-A-IP>

---
Note: Agar Instance A 'Ubuntu' tha toh path '/home/ubuntu/.ssh/' hoga.
