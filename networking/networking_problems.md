# 🌐 DevOps Networking — Scenario-Based Interview Q&A

> **7 Real-World Scenario Questions** | Commands + Concepts + Debugging Steps
> Prepared for: DevOps Interview Preparation

---

## Q1. 🔴 "Production mein ek pod kisi external API se connect nahi ho pa raha. Tum kya karoge?"

### Scenario:
Tumhara Kubernetes pod ek third-party payment API ko hit kar raha hai lekin `connection timed out` aa raha hai.

### Step-by-Step Approach:

**Step 1 — Pod ke andar jaao aur basic connectivity check karo**
```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# DNS resolution check
nslookup api.payment.com
dig api.payment.com

# Basic ping (agar allowed ho)
ping api.payment.com

# TCP port reachability check
curl -v --connect-timeout 5 https://api.payment.com
# ya
nc -zv api.payment.com 443
```

**Step 2 — Pod ka IP aur routing dekho**
```bash
ip addr show
ip route show
# Default gateway sahi hai?
```

**Step 3 — DNS resolution kaam kar raha hai?**
```bash
# CoreDNS check
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>

# resolv.conf check
cat /etc/resolv.conf
```

**Step 4 — Network Policy block to nahi kar rahi?**
```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>
```

**Step 5 — Node level pe check**
```bash
# Node se test karo
ssh <node-ip>
curl -v https://api.payment.com

# Security Group / Firewall rules check (AWS)
aws ec2 describe-security-groups --group-ids <sg-id>
```

### Root Causes (Common):
| Cause | Fix |
|-------|-----|
| NetworkPolicy blocking egress | Egress rule add karo |
| Security Group missing rule | Outbound 443 allow karo |
| NAT Gateway issue | Route table check karo |
| DNS not resolving | CoreDNS restart / configmap check |

---

## Q2. 🟡 "Two microservices ek doosre se baat nahi kar pa rahe cluster ke andar. Debug karo."

### Scenario:
`service-a` pod `service-b` ko `http://service-b:8080` pe call kar raha hai — `connection refused` aa raha hai.

### Step-by-Step Approach:

**Step 1 — Service exist karti hai?**
```bash
kubectl get svc -n <namespace>
kubectl describe svc service-b -n <namespace>
# Endpoints dekho — khaali hai to pods ready nahi hain
kubectl get endpoints service-b -n <namespace>
```

**Step 2 — Target pods running hain?**
```bash
kubectl get pods -n <namespace> -l app=service-b
kubectl describe pod <service-b-pod> -n <namespace>
# CrashLoopBackOff? ImagePullBackOff? OOMKilled?
```

**Step 3 — Port sahi hai?**
```bash
# Service ka port aur pod ka containerPort match karte hain?
kubectl get svc service-b -o yaml | grep -A5 "ports:"
kubectl get pod <pod> -o yaml | grep -A5 "containerPort"
```

**Step 4 — Service-A ke pod se direct test**
```bash
kubectl exec -it <service-a-pod> -n <namespace> -- /bin/sh
curl http://service-b:8080/health
# DNS name se na chale to ClusterIP se try karo
curl http://<cluster-ip>:8080/health
```

**Step 5 — kube-proxy check**
```bash
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod> | tail -50
```

### Key Concept:
> Kubernetes Service DNS format: `<service-name>.<namespace>.svc.cluster.local`
> Short form sirf same namespace mein kaam karta hai.

---

## Q3. 🟠 "Node suddenly unreachable ho gaya — `kubectl get nodes` pe NotReady aa raha hai. Kya steps loge?"

### Scenario:
Production cluster mein ek worker node `NotReady` state mein hai. Pods evict ho rahe hain.

### Step-by-Step Approach:

**Step 1 — Node status details dekho**
```bash
kubectl describe node <node-name>
# "Conditions" section dekho — MemoryPressure? DiskPressure? NetworkUnavailable?
```

**Step 2 — Node pe SSH karo**
```bash
ssh ec2-user@<node-ip>  # ya AWS SSM se
```

**Step 3 — kubelet service check karo**
```bash
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager
# Restart karo agar stopped hai
sudo systemctl restart kubelet
```

**Step 4 — Disk aur memory pressure check**
```bash
df -h          # Disk full?
free -m        # Memory?
top            # High CPU process?
```

**Step 5 — Container runtime check**
```bash
sudo systemctl status containerd   # ya docker
sudo crictl ps                     # containers running hain?
```

**Step 6 — Network interfaces check**
```bash
ip addr show
ping <master-node-ip>
# CNI plugin sahi chal raha hai?
ls /etc/cni/net.d/
```

**Step 7 — Agar fix nahi hua — node drain karo**
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# AWS mein — instance terminate karke Auto Scaling Group naya launch kare
```

---

## Q4. 🔵 "Users ko website slow lag rahi hai. Tum network angle se kaise investigate karoge?"

### Scenario:
Monitoring alert aaya — P99 latency 2s cross kar gayi. Tumhe network layer se debug karna hai.

### Step-by-Step Approach:

**Step 1 — Kahan slow hai? (Client ya Server)**
```bash
# Apne machine se
curl -w "\n\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://yourapp.com
```

**Step 2 — DNS resolution time zyada hai?**
```bash
time dig yourapp.com
# Agar zyada time lag raha hai — DNS provider ya Route53 check karo
```

**Step 3 — Load Balancer logs check**
```bash
# AWS ALB access logs (S3 se)
aws s3 cp s3://<alb-logs-bucket>/... /tmp/alb.log
grep "5[0-9][0-9] " /tmp/alb.log | tail -50

# ALB target response time dekho
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=<alb-arn> \
  --start-time 2024-01-01T00:00:00 --end-time 2024-01-01T01:00:00 \
  --period 60 --statistics Average
```

**Step 4 — Pod level latency**
```bash
# Prometheus se (agar setup hai)
# PromQL query:
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Step 5 — Network bandwidth saturation?**
```bash
# Node pe
iftop -i eth0        # real-time traffic
nethogs              # per-process bandwidth
ss -s                # socket statistics
```

**Step 6 — Packet loss check**
```bash
mtr --report yourapp.com   # traceroute + ping combined
ping -c 100 <target-ip> | tail -5
```

---

## Q5. 🟣 "Naya service deploy kiya — bahar se accessible nahi hai. Ingress ya Load Balancer issue debug karo."

### Scenario:
`kubectl apply` kar diya — pod running hai, lekin `https://app.example.com` pe `502 Bad Gateway` aa raha hai.

### Step-by-Step Approach:

**Step 1 — Pod khud healthy hai?**
```bash
kubectl get pods -n <namespace>
kubectl logs <pod-name> -n <namespace>
# Port 8080 pe listen kar raha hai?
kubectl exec -it <pod> -- netstat -tlnp
# ya
kubectl exec -it <pod> -- ss -tlnp
```

**Step 2 — Service endpoints sahi hain?**
```bash
kubectl get svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>
# Endpoints khaali? — selector mismatch hai
kubectl describe svc <service-name> -n <namespace> | grep Selector
kubectl get pods -n <namespace> --show-labels
```

**Step 3 — Ingress check karo**
```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
# Backend service name aur port sahi hain?
# TLS certificate configured hai?
```

**Step 4 — Ingress Controller logs**
```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <ingress-controller-pod> | grep "example.com"
# 502 = upstream pod se connection nahi
# 404 = path match nahi hua
```

**Step 5 — AWS Load Balancer health check**
```bash
# Target Group mein instances healthy hain?
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>
# Unhealthy? — health check path aur port verify karo
```

**Step 6 — Security Group**
```bash
# ALB se pod tak traffic allow hai?
aws ec2 describe-security-groups --group-ids <node-sg-id>
# Inbound: ALB security group se port 8080 allow hona chahiye
```

---

## Q6. 🟤 "Terraform se VPC banaya — EC2 instances internet se connect nahi ho rahe. Kya check karoge?"

### Scenario:
Private subnet mein EC2 hai, NAT Gateway laga diya phir bhi `yum update` timeout ho raha hai.

### Step-by-Step Approach:

**Step 1 — Subnet type check — Public hai ya Private?**
```bash
# AWS Console ya CLI se
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query 'Subnets[*].{ID:SubnetId,Public:MapPublicIpOnLaunch,AZ:AvailabilityZone}'
```

**Step 2 — Route Table check**
```bash
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values=<subnet-id>

# Private subnet ka route table mein hona chahiye:
# Destination: 0.0.0.0/0  -->  Target: nat-xxxxxxxx
# Public subnet mein:
# Destination: 0.0.0.0/0  -->  Target: igw-xxxxxxxx
```

**Step 3 — NAT Gateway status**
```bash
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available

# NAT Gateway PUBLIC subnet mein hona chahiye!
# Elastic IP attached hai?
```

**Step 4 — Security Group outbound rules**
```bash
aws ec2 describe-security-groups --group-ids <ec2-sg-id> \
  --query 'SecurityGroups[*].IpPermissionsEgress'

# Outbound: 0.0.0.0/0 allow hona chahiye
```

**Step 5 — NACL (Network ACL) check**
```bash
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=<subnet-id>

# Outbound rule 0.0.0.0/0 ALLOW?
# Inbound ephemeral ports (1024-65535) ALLOW? (return traffic ke liye)
```

**Step 6 — EC2 pe test**
```bash
ssh ec2-user@<private-ip>  # bastion ya SSM se
curl -v https://google.com
ip route show   # default route kya hai?
```

### Common Terraform Mistake:
```hcl
# ❌ WRONG — NAT Gateway private subnet mein daal diya
resource "aws_nat_gateway" "main" {
  subnet_id = aws_subnet.private.id  # GALAT!
}

# ✅ CORRECT — NAT Gateway PUBLIC subnet mein hona chahiye
resource "aws_nat_gateway" "main" {
  subnet_id     = aws_subnet.public.id   # SAHI
  allocation_id = aws_eip.nat.id
}
```

---

## Q7. 🔶 "Kubernetes cluster mein DNS resolution fail ho rahi hai — pods ko service names resolve nahi ho rahe. Debug karo."

### Scenario:
Application logs mein `dial tcp: lookup service-b on 10.96.0.10:53: no such host` aa raha hai.

### Step-by-Step Approach:

**Step 1 — CoreDNS pods healthy hain?**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl describe pod <coredns-pod> -n kube-system
kubectl logs <coredns-pod> -n kube-system
```

**Step 2 — CoreDNS service ka ClusterIP sahi hai?**
```bash
kubectl get svc kube-dns -n kube-system
# ClusterIP: 10.96.0.10 hona chahiye (default)

# Pod ka resolv.conf check karo
kubectl exec -it <any-pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10 hona chahiye
```

**Step 3 — Pod ke andar se DNS test karo**
```bash
kubectl run dns-test --image=busybox --restart=Never --rm -it -- /bin/sh

# Phir andar se:
nslookup kubernetes.default.svc.cluster.local
nslookup service-b.default.svc.cluster.local
nslookup google.com   # external DNS bhi kaam kar raha hai?
```

**Step 4 — CoreDNS ConfigMap check**
```bash
kubectl get configmap coredns -n kube-system -o yaml

# Expected Corefile:
# .:53 {
#   kubernetes cluster.local in-addr.arpa ip6.arpa {
#     pods insecure
#     fallthrough in-addr.arpa ip6.arpa
#   }
#   forward . /etc/resolv.conf
# }
```

**Step 5 — CoreDNS restart karo**
```bash
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system
```

**Step 6 — Node ka DNS check**
```bash
# Agar node ka DNS hi broken hai
ssh <node>
cat /etc/resolv.conf
nslookup google.com
systemd-resolve --status
```

**Step 7 — NetworkPolicy CoreDNS ko block to nahi kar rahi?**
```bash
kubectl get networkpolicy -n <namespace>
# Pod ko port 53 (UDP/TCP) pe kube-system se traffic allow hona chahiye

# Fix — egress allow karo DNS ke liye:
# ports:
# - port: 53
#   protocol: UDP
# - port: 53
#   protocol: TCP
```

### DNS Troubleshooting Cheatsheet:
| Symptom | Likely Cause |
|---------|-------------|
| Internal names resolve nahi | CoreDNS crash / NetworkPolicy |
| External names resolve nahi | CoreDNS forward config / Node DNS |
| Intermittent failures | CoreDNS resource limits / conntrack table full |
| Slow resolution | ndots setting (5 by default — zyada lookups) |

---

## 📌 Quick Reference — Important Commands

```bash
# Network debugging toolkit
kubectl exec -it <pod> -- /bin/sh
curl -v <url>
nc -zv <host> <port>
nslookup <hostname>
dig <hostname>
ip route show
ss -tlnp
netstat -tlnp
iftop -i eth0
mtr <host>

# Kubernetes network
kubectl get networkpolicy -A
kubectl get endpoints -A
kubectl get ingress -A
kubectl describe svc <name>

# AWS Network
aws ec2 describe-route-tables
aws ec2 describe-security-groups
aws ec2 describe-nat-gateways
aws elbv2 describe-target-health
```

---

*Document Version: 1.0 | DevOps Interview Prep — Networking Focus*
