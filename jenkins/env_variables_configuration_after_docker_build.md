# 🚀 DOCKER ENV VARIABLE INJECTION — DEEP DIVE (STEP-BY-STEP, PRACTICAL)

---

# 🧠 1. CORE IDEA (SABSE IMPORTANT)

👉 App ko values chahiye hoti hain (DB, API, PORT etc.)

❌ Hum `.env` file Git me push nahi karte  
✅ Isliye hum **runtime pe values inject karte hain**

---

# 🔥 2. DOCKER ME 2 PHASE HOTE HAIN

```
BUILD TIME  → docker build
RUN TIME    → docker run
```

---

# ⚙️ 3. BUILD TIME (KYA HOTA HAI?)

👉 Image ban rahi hoti hai  
👉 App run nahi ho raha hota  

👉 Yahan:
❌ env variables normally available nahi hote

---

## ❌ Example

Dockerfile:

```
RUN echo $DB_HOST
```

👉 Output:
```
(blank)
```

👉 Kyun?
👉 Kyunki DB_HOST define hi nahi hai build ke time

---

# 🚀 4. RUN TIME (REAL GAME YAHAN HAI)

👉 Container start hota hai  
👉 App run hota hai  

👉 Yahan:
✅ env variables inject karte hain

---

## ✅ Example

```
docker run -e DB_HOST=prod-db my-app
```

👉 Container ke andar:

```
process.env.DB_HOST = prod-db
```

---

# 🧩 5. MULTIPLE VARIABLES

```
docker run -e DB_HOST=prod -e PORT=3000 my-app
```

---

# 📂 6. .env FILE SE

```
docker run --env-file .env my-app
```

👉 `.env` example:

```
DB_HOST=prod-db
PORT=3000
```

---

# 🐳 7. DOCKERFILE ME ENV

---

## 🔹 Default value dena

```
ENV DB_HOST=localhost
```

👉 Agar kuch pass nahi kiya:

```
DB_HOST = localhost
```

---

## 🔹 Override karna

```
docker run -e DB_HOST=prod-db my-app
```

👉 Final:

```
DB_HOST = prod-db
```

---

# ⚠️ 8. BUILD ARG (ADVANCED)

👉 Build time pe value dene ke liye

---

## Dockerfile

```
ARG DB_HOST
RUN echo $DB_HOST
```

---

## Command

```
docker build --build-arg DB_HOST=prod-db .
```

---

## ❗ Important

👉 ARG:
- sirf build time ke liye hota hai
- container me available nahi hota

---

# 🧠 9. ARG vs ENV

| Feature | ARG | ENV |
|--------|-----|-----|
| Build time | ✅ | ✅ |
| Run time | ❌ | ✅ |
| Use | build config | app config |

---

# 🔥 10. REAL FLOW

```
Code →
docker build →
Image →
docker run →
ENV inject →
App start
```

---

# 🧩 11. NODEJS REAL EXAMPLE

---

## app.js

```
const db = process.env.DB_HOST;
console.log(db);
```

---

## Run

```
docker run -e DB_HOST=prod-db node-app
```

---

## Output

```
prod-db
```

---

# 🐳 12. COMPLETE DOCKERFILE

```
FROM node:18

WORKDIR /app
COPY . .

RUN npm install

ENV PORT=3000

CMD ["node", "app.js"]
```

---

# 🔐 13. SECURITY BEST PRACTICES

---

## ❌ Galat

```
ENV PASSWORD=123456
```

---

## ✅ Sahi

```
docker run -e PASSWORD=secure-value my-app
```

---

## ✅ Aur better

👉 Secrets use karo:
- Jenkins credentials
- Kubernetes secrets
- AWS Secrets Manager

---

# ☸️ 14. KUBERNETES CONNECTION

---

## ConfigMap

```
env:
  - name: DB_HOST
    value: "prod-db"
```

---

## Secret

```
env:
  - name: DB_PASS
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

---

# 🧠 15. KYUN RUNTIME ENV?

👉 Same image har jagah use ho sake

```
Dev → DB_HOST=dev-db
Prod → DB_HOST=prod-db
```

👉 Image same, config alag

---

# 💥 16. COMMON MISTAKES

❌ Build time pe env expect karna  
❌ Dockerfile me secrets likhna  
❌ `.env` push kar dena  

---

# 🚀 17. DEBUGGING

---

## Check env

```
docker exec -it <container> env
```

---

## Print

```
echo $DB_HOST
```

---

# 🧠 18. ANALOGY

```
docker build = ghar banana 🏠
docker run = ghar me rehna 🧑

env = furniture 🪑

👉 ghar banate waqt nahi hota
👉 rehne ke time lagta hai
```

---

# 🎯 19. FINAL SUMMARY

👉 docker build → image banata hai  
👉 docker run → env inject karta hai  

👉 ARG → build ke liye  
👉 ENV → runtime ke liye  

---

# 🔥 GOLD LINE

```
Build once → Run anywhere → Inject env at runtime
```

---
