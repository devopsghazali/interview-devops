# 🚀 COMPLETE DEVOPS FLOW (MAVEN + JAR + NEXUS + CI/CD) — PRACTICAL GUIDE

---

# 🧠 1. MAVEN KYA HAI?

Maven ek **build automation tool** hai jo Java projects ko manage karta hai.

👉 Simple language:
Maven = tool jo code ko **run hone layak bana deta hai**

👉 Kaam:
- Code compile
- Dependencies download
- Test run
- JAR/WAR banana

---

# 📂 2. MAVEN PROJECT STRUCTURE

project/
 ├── src/
 │   ├── main/java/      → actual code
 │   ├── test/java/      → test cases
 ├── pom.xml             → sabse important file
 └── target/             → output yahin aata hai

---

# ⚙️ 3. MAVEN COMMANDS (BUILD TARGETS)

Maven ek fixed flow follow karta hai:

clean → compile → test → package → install → deploy

---

## 🔹 Commands samajh lo ek-ek karke

### 1️⃣ mvn clean
👉 target folder delete karta hai

---

### 2️⃣ mvn compile
👉 .java → .class

---

### 3️⃣ mvn test
👉 test cases run

---

### 4️⃣ mvn package
👉 JAR file banata hai

---

### 5️⃣ mvn install
👉 local machine me save karta hai (~/.m2)

---

### 6️⃣ mvn deploy
👉 remote repo (Nexus) me upload karta hai

---

# 🔥 IMPORTANT RULE

Agar tum likhte ho:

mvn package

👉 Maven automatically karega:
compile + test + package

---

# 📦 4. JAR FILE KYA HAI?

👉 JAR = Java Archive

👉 Ye ek **zip file hoti hai jisme:**
- compiled code
- config
- resources

---

## 📂 Example

app.jar
 ├── META-INF/
 ├── classes/
 ├── application.properties

---

# ⚔️ NODE vs JAVA

Node:
npm run build → dist/

Java:
mvn package → target/app.jar

👉 SAME CONCEPT:
dist = folder  
jar = single file bundle

---

# 🚀 5. CI/CD FLOW (REAL)

Code Push →
Build →
Test →
Security →
Package →
Store →
Deploy

---

# 🧩 6. GITHUB ACTIONS ME USE

Basic example:

steps:
  - uses: actions/checkout@v4
  - run: mvn clean package

---

# 📦 7. ARTIFACT KYA HAI?

👉 Final output

Examples:
- jar
- war
- zip
- docker image

---

# 🏢 8. NEXUS KYA HAI?

👉 Nexus = artifact storage system

👉 Matlab:
JAR ko ek central jagah pe store karna

---

# 🚀 9. JAR KO NEXUS ME STORE KARNE KE TAREEKE

---

## 🔹 METHOD 1: MAVEN DEPLOY (BEST)

STEP 1: pom.xml me likho

<distributionManagement>
  <repository>
    <id>nexus</id>
    <url>http://your-nexus-url/repository/maven-releases/</url>
  </repository>
</distributionManagement>

---

STEP 2: settings.xml

~/.m2/settings.xml

<servers>
  <server>
    <id>nexus</id>
    <username>admin</username>
    <password>password</password>
  </server>
</servers>

---

STEP 3: run command

mvn clean deploy

👉 RESULT:
JAR Nexus me chala jayega

---

## 🔹 METHOD 2: CURL (MANUAL)

curl -u user:pass \
--upload-file target/app.jar \
http://nexus-url/repository/maven-releases/app.jar

---

## 🔹 METHOD 3: NEXUS UI

- login
- upload
- file select
- done

---

## 🔹 METHOD 4: CI/CD PIPELINE

GitHub Actions:

- run: mvn deploy

---

# 📦 10. ARTIFACT STORE KARNE KE OPTIONS

---

## 🟢 1. LOCAL

~/.m2/repository

👉 sirf testing ke liye

---

## 🟡 2. NEXUS

👉 sabse common
👉 enterprise use

---

## 🔵 3. ARTIFACTORY

👉 Nexus jaisa hi
👉 thoda advanced

---

## 🟣 4. AWS S3

👉 simple storage
👉 versioning possible

---

## 🔴 5. DOCKER REGISTRY

👉 docker images ke liye

---

## ⚫ 6. GITHUB PACKAGES

👉 GitHub ke andar hi storage

---

# 🧠 11. VERSIONING

Example:

app-1.0.jar  
app-1.1.jar  
app-2.0.jar  

👉 rollback easy

---

# 🔥 SNAPSHOT vs RELEASE

SNAPSHOT → testing  
RELEASE → production

---

# 🚀 12. REAL DEVOPS FLOW

Developer →
GitHub →
GitHub Actions →
Maven Build →
JAR →
Nexus →
Docker →
Registry →
Kubernetes

---

# 💥 13. DOCKER CONNECTION

Dockerfile:

COPY target/app.jar /app/app.jar
CMD ["java", "-jar", "app.jar"]

---

# 🎯 FINAL UNDERSTANDING

👉 Maven = build tool  
👉 Commands = steps  
👉 JAR = final app  
👉 Nexus = storage  
👉 CI/CD = automation  

---

# 🚀 END GOAL

Tumhe aana chahiye:

✔ Maven build  
✔ JAR banana  
✔ Nexus me store  
✔ Pipeline banana  
✔ Deploy karna  

---

# 🔥 GOLD LINE

mvn clean package → jar  
mvn deploy → nexus  
docker build → image  
kubectl apply → deploy

---

# 🧠 FINAL ANALOGY

Code = raw material  
Maven = factory  
JAR = product  
Nexus = warehouse  
Kubernetes = delivery system 🚚

---
