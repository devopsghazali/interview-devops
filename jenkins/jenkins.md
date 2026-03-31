# 🔧 Jenkins — Complete Interview Q&A (7 Scenarios)

> Ek baar padh lo — poora Jenkins cover ho jayega
> Concepts: Architecture, Freestyle vs Pipeline, Jenkinsfile, Agents, Shared Libraries, Credentials, Multibranch, Monitoring

---

## Q1. 🔴 "Jenkins ka Architecture explain karo aur first pipeline setup karo."

### Scenario:
Interviewer poochhe — Jenkins kaise kaam karta hai aur ek basic pipeline dikhao.

### Jenkins Architecture:

```
Jenkins Master (Controller)
├── Web UI (port 8080)
├── Job scheduler
├── Build queue
├── Plugin management
└── Credentials store

Jenkins Agents (Workers)
├── Agent 1 (Linux — builds)
├── Agent 2 (Linux — tests)
└── Agent 3 (Windows — .NET builds)

Communication: Master → Agent (SSH ya JNLP)
Master sirf orchestrate karta hai, actual work agent karta hai
```

### Jenkins Install (Docker mein):

```bash
# Jenkins locally chalao
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Initial password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### First Declarative Pipeline — Jenkinsfile:

```groovy
// Jenkinsfile (project root mein rakho)

pipeline {
    agent any   // kisi bhi available agent pe chalao

    // Environment variables
    environment {
        APP_NAME = 'my-app'
        DOCKER_REGISTRY = '123456789.dkr.ecr.ap-south-1.amazonaws.com'
    }

    // Build triggers
    triggers {
        pollSCM('H/5 * * * *')    // har 5 min mein Git check karo
        // ya webhook se trigger hota hai
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm   // Git se code lo
                echo "Building branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    // JUnit test results publish karo
                    junit 'test-results/**/*.xml'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }

    // Post actions — kisi bhi result pe
    post {
        success {
            echo '✅ Pipeline successful!'
        }
        failure {
            echo '❌ Pipeline failed!'
            // email ya Slack notification
        }
        always {
            cleanWs()    // workspace clean karo
        }
    }
}
```

### Declarative vs Scripted Pipeline:

```groovy
// Declarative (RECOMMENDED — structured, readable)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'make' }
        }
    }
}

// Scripted (flexible but complex — Groovy code)
node {
    stage('Build') {
        sh 'make'
    }
}
```

> **Key Concepts:** Master/Agent architecture | Declarative pipeline | Jenkinsfile | stages/steps/post | pollSCM

---

## Q2. 🟡 "Jenkins Agents configure karo — Docker agent aur dedicated node."

### Scenario:
Build environment polluted ho gayi hai — har build alag clean environment chahiye.

### Agent Types:

```groovy
pipeline {
    // Type 1: Any available agent
    agent any

    // Type 2: Specific label wala agent
    agent { label 'linux-docker' }

    // Type 3: Docker container as agent (BEST for isolation)
    agent {
        docker {
            image 'node:18-alpine'
            args '-v /tmp:/tmp'
            reuseNode false   // fresh container har build pe
        }
    }

    // Type 4: Kubernetes pod as agent
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:18-alpine
    command: ['cat']
    tty: true
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
'''
            defaultContainer 'node'
        }
    }
}
```

### Per-Stage Agent (different containers):

```groovy
pipeline {
    agent none   // global agent nahi chahiye

    stages {
        stage('Test') {
            agent {
                docker { image 'node:18-alpine' }
            }
            steps {
                sh 'npm test'
            }
        }

        stage('Build Image') {
            agent {
                docker {
                    image 'docker:24'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker build -t my-app .'
            }
        }

        stage('Deploy') {
            agent { label 'production-node' }   // specific server
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

### Permanent Agent Add karna (Node):

```bash
# Jenkins UI mein:
# Manage Jenkins → Nodes → New Node

# SSH se connect karo:
# Host: agent-server-ip
# Credentials: SSH key
# Remote directory: /home/jenkins

# Agent pe requirements:
# - Java installed hona chahiye
# - Jenkins user banana
# - Docker group mein add karna (agar Docker use karna hai)
sudo usermod -aG docker jenkins
```

### Agent Labels — Best Practice:

```groovy
// Labels se agent select karo
// linux && docker — dono conditions
// java || python — koi ek

pipeline {
    agent { label 'linux && docker && high-memory' }
}
```

> **Key Concepts:** Docker agent | Kubernetes agent | Per-stage agents | SSH node | agent none | Label expressions

---

## Q3. 🟠 "Credentials securely manage karo Jenkins mein — AWS, Docker, SSH, Git tokens."

### Scenario:
Developer ne Jenkinsfile mein AWS credentials hardcode kar diye — security breach ho gayi.

### Credentials Add karna:

```
Jenkins UI:
Manage Jenkins → Credentials → System → Global → Add Credentials

Types:
├── Username with password (Docker registry, Git)
├── Secret text (API keys, tokens)
├── SSH Username with private key
├── AWS Credentials (plugin se)
└── Certificate
```

### Credentials Use karna Jenkinsfile mein:

```groovy
pipeline {
    agent any

    environment {
        // Secret text — directly variable mein
        API_KEY = credentials('my-api-key')

        // Username/Password — 3 variables milenge
        // DOCKER_CREDS = username:password
        // DOCKER_CREDS_USR = username
        // DOCKER_CREDS_PSW = password
        DOCKER_CREDS = credentials('docker-registry-creds')
    }

    stages {
        stage('Docker Login') {
            steps {
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login \
                      -u $DOCKER_CREDS_USR \
                      --password-stdin registry.example.com
                '''
                // Password logs mein *** se mask hoga
            }
        }

        stage('AWS Deploy') {
            steps {
                // Method 1: withCredentials block
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh 'aws s3 ls'
                }
                // Block ke baad credentials automatically clear

                // Method 2: AWS Credentials Plugin
                withAWS(credentials: 'aws-prod', region: 'ap-south-1') {
                    sh 'aws eks update-kubeconfig --name my-cluster'
                }
            }
        }

        stage('SSH Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'prod-server-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
                          $SSH_USER@prod-server.example.com \
                          "cd /app && git pull && ./restart.sh"
                    '''
                }
            }
        }

        stage('Git Operations') {
            steps {
                withCredentials([
                    string(credentialsId: 'github-token', variable: 'GH_TOKEN')
                ]) {
                    sh 'git clone https://$GH_TOKEN@github.com/org/repo.git'
                }
            }
        }
    }
}
```

### Credentials Scopes:

```
Global    — sabhi jobs ke liye
System    — Jenkins internals ke liye (plugins etc)
Folder    — specific folder ke jobs ke liye (multi-team)
```

> **Key Concepts:** withCredentials | credentials() binding | Masking | Scope | AWS/SSH/Docker creds | Security best practices

---

## Q4. 🔵 "Multibranch Pipeline — Har branch ka alag environment, PR pe automatic build."

### Scenario:
10 developers hain, alag-alag feature branches pe kaam kar rahe hain — har branch ka CI chahiye.

### Multibranch Pipeline Setup:

```
Jenkins UI:
New Item → Multibranch Pipeline

Configuration:
├── Branch Sources: GitHub/GitLab
├── Credentials: Git token
├── Discover branches: All / matching pattern
├── Discover pull requests
└── Build strategies
```

### Jenkinsfile — Branch Aware Logic:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'npm ci && npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        // Docker push — sirf main/develop pe
        stage('Push Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                }
            }
            steps {
                sh './scripts/docker-push.sh'
            }
        }

        // Deploy Staging — sirf develop branch pe
        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh './scripts/deploy.sh staging'
            }
        }

        // Deploy Production — sirf main pe, manual approval ke baad
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                // Human approval
                input message: 'Production deploy karna hai?',
                      ok: 'Deploy Karo!',
                      submitter: 'admin,tech-lead'

                sh './scripts/deploy.sh production'
            }
        }
    }

    post {
        // PR pe comment add karo
        success {
            script {
                if (env.CHANGE_ID) {  // CHANGE_ID = PR number
                    pullRequest.createStatus(
                        status: 'success',
                        context: 'jenkins-ci',
                        description: 'Build passed!'
                    )
                }
            }
        }
    }
}
```

### When Conditions — Complete Guide:

```groovy
when {
    branch 'main'                           // exact branch match
    branch pattern: 'feature/.*'            // regex pattern
    tag 'v*'                                // tag match
    environment name: 'DEPLOY_ENV', value: 'production'
    expression { return params.DEPLOY == true }
    changeRequest()                         // sirf PR/MR pe
    not { branch 'main' }                   // negation
    allOf {                                 // AND condition
        branch 'main'
        environment name: 'ENV', value: 'prod'
    }
    anyOf {                                 // OR condition
        branch 'main'
        branch 'develop'
    }
}
```

> **Key Concepts:** Multibranch Pipeline | when conditions | input approval | CHANGE_ID | Branch patterns | PR builds

---

## Q5. 🟣 "Shared Libraries — Common code centralize karo, DRY principle."

### Scenario:
20 microservices hain — sab mein same Docker build, test, deploy Groovy code copy-paste hai. Centralize karo.

### Shared Library Structure:

```
jenkins-shared-library/           (alag Git repo)
├── vars/                         (Global variables / functions)
│   ├── dockerBuild.groovy
│   ├── deployToKubernetes.groovy
│   └── sendSlackNotification.groovy
├── src/                          (Groovy classes)
│   └── org/company/
│       └── DeployUtils.groovy
└── resources/                    (static files, scripts)
    └── scripts/deploy.sh
```

### vars/dockerBuild.groovy:

```groovy
// vars/dockerBuild.groovy
// call() method — pipeline mein function ki tarah use karo

def call(Map config = [:]) {
    // Default values
    def imageName = config.imageName ?: error("imageName required")
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def registry = config.registry ?: env.DOCKER_REGISTRY

    echo "Building Docker image: ${imageName}"

    docker.withRegistry("https://${registry}", config.credentialsId ?: 'docker-creds') {
        def image = docker.build(
            "${registry}/${imageName}:${env.BUILD_NUMBER}",
            "--file ${dockerfile} ."
        )
        image.push()
        image.push('latest')

        // Return image tag
        return "${registry}/${imageName}:${env.BUILD_NUMBER}"
    }
}
```

### vars/deployToKubernetes.groovy:

```groovy
def call(Map config = [:]) {
    def namespace = config.namespace ?: 'default'
    def deployment = config.deployment ?: error("deployment name required")
    def imageTag = config.imageTag ?: error("imageTag required")
    def timeout = config.timeout ?: '5m'

    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh """
            kubectl set image deployment/${deployment} \
              ${deployment}=${imageTag} \
              -n ${namespace}
            kubectl rollout status deployment/${deployment} \
              -n ${namespace} \
              --timeout=${timeout}
        """
    }
}
```

### Shared Library Register karna:

```
Jenkins UI:
Manage Jenkins → System → Global Pipeline Libraries

Name: company-shared-lib
Default version: main
Source: Git
URL: https://github.com/my-org/jenkins-shared-library.git
Credentials: github-token
```

### Use karna — Microservice ka Jenkinsfile:

```groovy
// Shared library import karo
@Library('company-shared-lib') _   // _ important hai

pipeline {
    agent any

    stages {
        stage('Build & Push') {
            steps {
                script {
                    // Ek line mein Docker build + push
                    def imageTag = dockerBuild(
                        imageName: 'service-a',
                        registry: '123456789.dkr.ecr.ap-south-1.amazonaws.com',
                        credentialsId: 'ecr-creds'
                    )
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Deploy') {
            steps {
                deployToKubernetes(
                    deployment: 'service-a',
                    namespace: 'production',
                    imageTag: env.IMAGE_TAG
                )
            }
        }
    }

    post {
        failure {
            sendSlackNotification(
                channel: '#deployments',
                message: "❌ ${env.JOB_NAME} failed!",
                webhookUrl: env.SLACK_WEBHOOK
            )
        }
    }
}
```

> **Key Concepts:** @Library | vars/ directory | call() method | Map config | Centralized logic | DRY

---

## Q6. 🟤 "Jenkins Pipeline fail ho gayi — Debug, Retry, aur Timeout handle karo."

### Scenario:
Flaky test hai jo kabhi kabhi fail hota hai — retry chahiye. Aur build hang ho jaati hai — timeout chahiye.

### Error Handling Patterns:

```groovy
pipeline {
    agent any

    // Global timeout — poori pipeline ke liye
    options {
        timeout(time: 1, unit: 'HOURS')
        retry(2)                         // poori pipeline 2 baar retry
        buildDiscarder(logRotator(numToKeepStr: '10'))  // sirf 10 builds rakho
        disableConcurrentBuilds()         // ek waqt ek hi build
        timestamps()                      // logs mein timestamp
    }

    stages {
        stage('Flaky Test') {
            options {
                retry(3)                  // sirf is stage ke liye retry
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                sh 'npm run flaky-test'   // 3 baar try karega
            }
        }

        stage('Deploy') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                // try-catch — error handle karo
                script {
                    try {
                        sh './deploy.sh'
                    } catch (Exception e) {
                        echo "Deploy failed: ${e.getMessage()}"

                        // Rollback karo
                        sh 'kubectl rollout undo deployment/my-app'

                        // Error propagate karo (build fail ho)
                        error("Deploy failed after rollback")
                    }
                }
            }
        }

        stage('Approval with Timeout') {
            steps {
                script {
                    // 1 ghante mein approve nahi kiya to abort
                    def result = input(
                        message: 'Deploy to production?',
                        ok: 'Deploy',
                        submitter: 'admin',
                        submitterParameter: 'APPROVED_BY',
                        parameters: [
                            choice(
                                name: 'ENVIRONMENT',
                                choices: ['staging', 'production'],
                                description: 'Where to deploy?'
                            )
                        ]
                    )
                    echo "Approved by: ${result.APPROVED_BY}"
                    echo "Deploying to: ${result.ENVIRONMENT}"
                }
            }
        }
    }
}
```

### Parallel Stages with Error Handling:

```groovy
stage('Parallel Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'npm run test:unit'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'npm run test:integration'
            }
        }
        stage('E2E Tests') {
            options { retry(2) }
            steps {
                sh 'npm run test:e2e'
            }
        }
    }
    // Ek fail ho to baki continue karein? (default: sab fail)
    // failFast: false  -- parallel block mein add karo
}
```

### Build Parameters:

```groovy
pipeline {
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Tests skip karo?')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Deploy where?')
    }

    stages {
        stage('Test') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            steps {
                sh 'npm test'
            }
        }

        stage('Deploy') {
            steps {
                sh "./deploy.sh ${params.ENVIRONMENT} ${params.IMAGE_TAG}"
            }
        }
    }
}
```

> **Key Concepts:** retry | timeout | try-catch | error() | parallel | input | parameters | buildDiscarder

---

## Q7. 🔶 "Jenkins monitoring, maintenance aur performance tuning karo."

### Scenario:
Jenkins slow ho gayi hai — builds queue mein pade hain, disk full hai, master pe load hai.

### Problem 1 — Disk Full:

```bash
# Jenkins master pe disk check
df -h /var/jenkins_home

# Purani builds cleanup karo
# UI se: Manage Jenkins → System → Build Record Root Directory

# Automated cleanup — Jenkinsfile mein
options {
    buildDiscarder(logRotator(
        numToKeepStr: '10',        # sirf 10 builds rakho
        daysToKeepStr: '30',       # 30 din se purani delete karo
        artifactNumToKeepStr: '5', # artifacts sirf 5 rakho
        artifactDaysToKeepStr: '7'
    ))
}

# Script Console se bulk cleanup (Manage Jenkins → Script Console):
def maxBuildsToKeep = 10
Jenkins.instance.getAllItems(Job.class).each { job ->
    def builds = job.getBuilds()
    if (builds.size() > maxBuildsToKeep) {
        builds[maxBuildsToKeep..-1].each { build ->
            build.delete()
        }
    }
}
```

### Problem 2 — Master Overloaded (Builds Master Pe Chal Rahe Hain):

```groovy
// ❌ GALAT — Master pe builds mat chalao
pipeline {
    agent any   // master pe bhi chal sakta hai

// ✅ SAHI — Dedicated agents use karo
pipeline {
    agent { label 'build-agent' }   // sirf labeled agents pe

// Master ko restrict karo:
// Manage Jenkins → Nodes → Built-In Node → # of executors = 0
```

### Problem 3 — Build Queue Bhara Hai:

```bash
# Queue status check karo
curl -u admin:token http://jenkins:8080/queue/api/json | python3 -m json.tool

# Agent add karo ya existing agents dekho
# Manage Jenkins → Nodes → Node status

# Docker agents auto-scale ke liye — kubernetes plugin
```

### Problem 4 — JVM Memory Issue:

```bash
# Jenkins Java options set karo
# /etc/default/jenkins ya Docker mein:
JAVA_OPTS="-Xmx2g -Xms512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -Djava.awt.headless=true"

# Docker mein:
docker run \
  -e JAVA_OPTS="-Xmx2g -Xms512m" \
  jenkins/jenkins:lts
```

### Jenkins Health Check — Groovy Script:

```groovy
// Manage Jenkins → Script Console

// Stuck builds dekho
Jenkins.instance.getAllItems(Job.class).each { job ->
    job.getBuilds().findAll { it.isBuilding() }.each { build ->
        def duration = System.currentTimeMillis() - build.getStartTimeInMillis()
        if (duration > 3600000) { // 1 hour se zyada
            println "STUCK: ${job.name} #${build.number} - ${duration/60000} minutes"
        }
    }
}

// Sab offline agents dekho
Jenkins.instance.nodes.each { node ->
    if (!node.toComputer().isOnline()) {
        println "OFFLINE: ${node.name}"
    }
}
```

### Jenkins Backup Strategy:

```bash
# Important files backup karo
# /var/jenkins_home/
# ├── config.xml              (main config)
# ├── credentials.xml         (credentials)
# ├── jobs/*/config.xml       (job configs)
# ├── plugins/                (installed plugins)
# └── secrets/                (encryption keys)

# ThinBackup Plugin (UI se configure karo)
# Ya manual:
tar czf jenkins-backup-$(date +%Y%m%d).tar.gz \
  --exclude='jobs/*/builds' \    # build history skip karo
  --exclude='workspace' \
  /var/jenkins_home/

# S3 pe upload
aws s3 cp jenkins-backup-*.tar.gz s3://my-backup-bucket/jenkins/
```

### Performance Optimization Checklist:

```
✅ Master executors = 0 (no builds on master)
✅ Sufficient agents configured
✅ Build log rotation set
✅ Old workspace cleanup (Workspace Cleanup Plugin)
✅ JVM heap properly set (Xmx = RAM ka 50%)
✅ Unused plugins remove karo
✅ GitHub webhook use karo (pollSCM nahi)
✅ Pipeline cache configure (node_modules etc)
✅ Parallel stages jahan possible
✅ Regular backup S3 pe
```

### Key Plugins — Must Know:

```
Pipeline               — Declarative pipeline support
Blue Ocean             — Modern UI
Git / GitHub           — SCM integration
Docker Pipeline        — Docker agent support
Kubernetes             — K8s agent
AWS Credentials        — AWS creds
Credentials Binding    — withCredentials
Slack Notification     — Slack alerts
JUnit                  — Test results
HTML Publisher         — Reports publish
Workspace Cleanup      — Workspace clean
ThinBackup             — Backup
OWASP Dependency-Check — Security scan
```

> **Key Concepts:** Disk cleanup | Master executors=0 | JVM tuning | Backup | Plugin management | Monitoring | Webhook vs pollSCM

---

## 📌 Jenkins — Quick Reference Map

```
Jenkins Pipeline Anatomy
├── pipeline {}
│   ├── agent {}              kahan chalega
│   ├── environment {}        env variables
│   ├── options {}            timeouts, retry, logs
│   ├── parameters {}         user inputs
│   ├── triggers {}           pollSCM, cron
│   └── stages {}
│       └── stage('Name') {}
│           ├── agent {}      per-stage override
│           ├── when {}       conditions
│           ├── options {}    per-stage options
│           ├── steps {}      actual commands
│           └── post {}       after steps
└── post {}                   after all stages

Important env Variables
├── env.BUILD_NUMBER          build number
├── env.JOB_NAME              job name
├── env.BRANCH_NAME           branch (multibranch)
├── env.WORKSPACE             workspace path
├── env.CHANGE_ID             PR number (multibranch)
├── env.BUILD_URL             link to this build
└── env.GIT_COMMIT            git commit hash

Step Conditions (when)
├── branch 'main'
├── tag 'v*'
├── changeRequest()
├── expression { groovy }
├── environment name:, value:
├── not { }
├── allOf { }
└── anyOf { }
```

---

*Jenkins Interview Prep v1.0 | Concepts: Architecture → Pipeline → Agents → Credentials → Multibranch → Shared Libs → Debug → Monitoring*
