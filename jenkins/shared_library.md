# 🚀 Jenkins Shared Library – Complete Guide (Beginner → Advanced)

---

# 📌 1. Introduction

**Jenkins Shared Library** ek aisa mechanism hai jo aapko **reusable pipeline code likhne aur multiple projects mein use karne** ki facility deta hai.

👉 Matlab:

* Same pipeline logic baar-baar likhne ki zarurat nahi
* Ek jagah likho → sab jagah use karo

---

# 🧠 2. Core Concept (Sabse Important)

```text
Shared Library = Reusable + Configurable Pipeline Logic
```

👉 Yeh “exact same commands” nahi hota
👉 Yeh “same process + different inputs” hota hai

---

# 🎯 3. Problem Statement (Why Shared Library?)

## ❌ Without Shared Library:

* Har project ka alag Jenkinsfile
* Duplicate code
* Maintain karna mushkil

## ✅ With Shared Library:

* Centralized logic
* Easy updates
* Clean pipelines

---

# 🧱 4. Shared Library Structure (Deep Understanding)

```
(shared-library-repo)
│
├── vars/
│   └── buildApp.groovy
│
├── src/
│   └── com/example/Helper.groovy
│
├── resources/
│
└── README.md
```

---

## 🔵 4.1 `vars/` → Entry Point (User-facing functions)

👉 Yeh wo jagah hai jahan se Jenkinsfile functions call karta hai

### Example:

```groovy
// vars/buildApp.groovy
def call() {
    echo "Building app..."
}
```

👉 Use in Jenkinsfile:

```groovy
buildApp()
```

---

## 🔵 4.2 `src/` → Logic Engine (Backend)

👉 Complex logic aur reusable classes yahan likhte hain

### Example:

```groovy
package com.example

class Helper {
    def sayHello() {
        return "Hello World"
    }
}
```

👉 Use via vars:

```groovy
import com.example.Helper

def call() {
    def h = new Helper()
    echo h.sayHello()
}
```

---

## 🔵 4.3 `resources/` → Config/Data Store

👉 Static files store karne ke liye:

* YAML
* JSON
* Shell scripts
* Templates

---

### Example: YAML Config

```yaml
# resources/config.yaml
appName: my-app
version: v1
```

---

### Use in pipeline:

```groovy
def configText = libraryResource 'config.yaml'
def config = readYaml text: configText

echo config.appName
```

---

## 🔵 4.4 README.md

👉 Documentation ke liye (human use only)

---

# 🔥 5. Golden Concept

```text
vars = kya karna hai
src = kaise karna hai
resources = kis data ke saath karna hai
```

---

# 🔧 6. Jenkins UI Setup (Step-by-Step)

## Step 1:

```
Manage Jenkins → Configure System
```

## Step 2:

Scroll to:

```
Global Pipeline Libraries
```

---

## 🔹 Configuration Fields Explained

### 1. Name

* Example: `my-lib`
* Used in Jenkinsfile

---

### 2. Default Version

* Git branch (main/dev)

---

### 3. Load Implicitly

* ON → @Library likhne ki zarurat nahi
* OFF → recommended (clear usage)

---

### 4. Allow Override

* Allows:

```groovy
@Library('my-lib@dev') _
```

---

### 5. Include Changes

* Library changes ko job history mein show karta hai

---

### 6. Cache on Controller

* Faster builds
* Recommended ON

---

### 7. Retrieval Method

* Always use:

```
Modern SCM
```

---

### 8. SCM → Git

* Repository URL provide karo

---

### 9. Credentials

* Private repo ke liye required

---

### 10. Discover Branches

* Multiple branches support

---

### 11. Fresh Clone per Build

* ON → fresh code (slow)
* OFF → faster (recommended)

---

### 12. Library Path

* Subfolder ke liye (optional)

---

# 🧩 7. `@Library` Symbol (Important)

## Basic Syntax:

```groovy
@Library('my-lib') _
```

---

## Breakdown:

| Part       | Meaning                      |
| ---------- | ---------------------------- |
| `@Library` | library load karo            |
| `'my-lib'` | library name                 |
| `_`        | just load, assign nahi karna |

---

## ❗ Without @Library:

```
No such DSL method error
```

---

## Advanced:

```groovy
@Library('my-lib@dev') _
```

---

# 🔁 8. Full Execution Flow

```
Jenkinsfile start
   ↓
@Library load
   ↓
Git repo clone
   ↓
vars/ scan
   ↓
functions register
   ↓
pipeline execute
```

---

# 🚀 9. Real Scenarios

---

## ✅ Scenario 1: Multi-Tech Build

```yaml
type: java
buildCmd: mvn clean install
```

👉 Same function → different tech

---

## ✅ Scenario 2: Environment Deployment

```yaml
env: prod
server: prod.example.com
```

👉 Same deploy logic → different environment

---

## ✅ Scenario 3: Docker Automation

```json
{
  "repo": "my-app",
  "tag": "v1"
}
```

👉 Dynamic Docker builds

---

## ✅ Scenario 4: Full Pipeline

```groovy
pipeline()
```

👉 Entire CI/CD abstracted

---

# ⚠️ 10. Common Mistakes

❌ Library name mismatch
❌ Wrong branch
❌ Missing credentials
❌ YAML parse na karna
❌ vars mein `call()` missing

---

# 🔥 11. Best Practices

✔ Small reusable functions likho
✔ Config ko resources mein rakho
✔ Hardcoding avoid karo
✔ Version control use karo
✔ Naming clear rakho

---

# 🎯 12. Final Summary

```text
Shared Library = Centralized CI/CD brain
```

👉 Ek jagah logic
👉 Har jagah execution

---

# 🧠 FINAL UNDERSTANDING

```
Jenkinsfile = trigger
@Library = gate
vars = interface
src = engine
resources = data
```

---

# 🚀 Conclusion

Agar tum Shared Library samajh gaye, toh:

* Tum scalable pipelines bana sakte ho
* Enterprise-level DevOps design kar sakte ho
* Aur repetitive kaam eliminate kar sakte ho

---

**🔥 Yeh Jenkins ka sabse powerful feature hai — mastery yahin se start hoti hai.**
