# ⚙️ GitHub Actions — Complete Interview Q&A (7 Scenarios)

> Ek baar padh lo — poora GitHub Actions cover ho jayega
> Concepts: Workflows, Triggers, Jobs, Steps, Secrets, Matrix, Caching, Reusable Workflows, Environments

---

## Q1. 🔴 "Pehli baar CI pipeline banao — PR pe automatically test run ho."

### Scenario:
Node.js app hai — har pull request pe lint, test aur build automatically chalana hai.

### Basic Workflow Structure:

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

# Trigger — Kab chalega?
on:
  pull_request:
    branches: [main, develop]    # sirf in branches ki PR pe
    paths:                        # sirf in files ke change pe (optional)
      - 'src/**'
      - 'package*.json'
  push:
    branches: [main]

# Jobs — Kya karna hai?
jobs:
  test:
    name: Lint, Test & Build
    runs-on: ubuntu-latest        # Runner type

    steps:
      # Step 1: Code checkout
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Node.js setup
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'            # npm cache automatic

      # Step 3: Dependencies install
      - name: Install dependencies
        run: npm ci               # npm install se behtar CI mein

      # Step 4: Lint
      - name: Run ESLint
        run: npm run lint

      # Step 5: Tests
      - name: Run tests
        run: npm test -- --coverage

      # Step 6: Build
      - name: Build application
        run: npm run build

      # Step 7: Upload coverage (optional)
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        if: always()              # test fail ho tab bhi upload karo
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7
```

### Workflow Concepts:

```yaml
# TRIGGERS (on:)
on:
  push:                    # code push pe
  pull_request:            # PR pe
  schedule:                # cron schedule
    - cron: '0 2 * * *'   # roz raat 2 baje
  workflow_dispatch:       # manual trigger (button se)
  workflow_call:           # doosre workflow se call karo (reusable)
  release:
    types: [published]     # release publish pe

# RUNNERS
runs-on: ubuntu-latest     # Linux (sabse common)
runs-on: windows-latest    # Windows
runs-on: macos-latest      # macOS
runs-on: self-hosted       # apna server

# EXPRESSIONS
${{ github.sha }}          # commit hash
${{ github.ref }}          # branch name
${{ github.actor }}        # user jisne push kiya
${{ github.event_name }}   # trigger event name
```

> **Key Concepts:** Workflow structure | on triggers | runs-on | steps vs jobs | actions/checkout | npm ci

---

## Q2. 🟡 "CI slow hai — 8 minutes lag rahe hain. Speed up karo caching se."

### Scenario:
`npm install` har baar fresh install karta hai — 4 minutes waste ho rahe hain.

### Caching Strategy:

```yaml
name: Optimized CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Method 1: setup-node ka built-in cache (recommended)
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'   # package-lock.json hash se cache key

      # Method 2: Manual cache control
      - name: Cache node_modules
        uses: actions/cache@v4
        id: npm-cache
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-   # fallback — partial match

      - name: Install (only if cache miss)
        if: steps.npm-cache.outputs.cache-hit != 'true'
        run: npm ci

      - run: npm test
```

### Docker Layer Caching in Actions:

```yaml
      # Docker build cache — bahut speed up karta hai
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=gha           # GitHub Actions cache se
          cache-to: type=gha,mode=max    # cache mein save karo
          tags: my-app:${{ github.sha }}
```

### Multiple Cache Types:

```yaml
      # Python pip cache
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      # Go modules cache
      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'
          cache: true

      # Custom path cache (koi bhi folder)
      - uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
```

### Parallel Jobs se Speed Up:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]    # lint aur test ke baad chalega
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
```

> **Key Concepts:** actions/cache | cache key | hashFiles | restore-keys fallback | Docker layer cache | Parallel jobs

---

## Q3. 🟠 "Secrets aur Environment Variables securely manage karo — Dev/Staging/Prod alag-alag."

### Scenario:
Production ka DB password galti se logs mein print ho gaya. Proper secrets management setup karo.

### Secrets Setup:

```yaml
# GitHub mein secrets add karo:
# Settings → Secrets and variables → Actions → New repository secret

name: Deploy Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Secrets use karna
      - name: Deploy to server
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}      # repo secret
          API_KEY: ${{ secrets.PROD_API_KEY }}
        run: |
          echo "Deploying..."
          # $DB_PASSWORD automatically masked in logs: ***

      # AWS credentials
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
```

### Environments — Dev/Staging/Prod:

```yaml
# GitHub Settings → Environments mein banao:
# development, staging, production
# Production pe: required reviewers add karo (approval chahiye)

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging    # staging environment ke secrets use hote hain
    steps:
      - name: Deploy
        env:
          DB_URL: ${{ secrets.DB_URL }}   # staging ka DB_URL
        run: ./deploy.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com   # deployment URL (optional, badge pe dikhta hai)
    steps:
      - name: Deploy
        env:
          DB_URL: ${{ secrets.DB_URL }}   # production ka DB_URL (alag value)
        run: ./deploy.sh production
```

### Variables vs Secrets:

```yaml
# Variables — non-sensitive config (visible in logs)
# Settings → Variables
${{ vars.API_BASE_URL }}      # visible
${{ vars.NODE_ENV }}

# Secrets — sensitive (masked in logs)
# Settings → Secrets
${{ secrets.DB_PASSWORD }}    # *** masked
${{ secrets.JWT_SECRET }}

# GITHUB_TOKEN — automatic (extra secrets nahi chahiye)
${{ secrets.GITHUB_TOKEN }}   # auto-provided, PRs/issues/packages ke liye
```

### Security Best Practices:

```yaml
      # ❌ GALAT — secret print ho jayega
      - run: echo "Password is ${{ secrets.DB_PASSWORD }}"

      # ✅ SAHI — env mein pass karo
      - name: Use secret
        env:
          MY_SECRET: ${{ secrets.DB_PASSWORD }}
        run: |
          # MY_SECRET masked rahega logs mein
          ./script.sh

      # ❌ GALAT — untrusted input directly mein
      - run: echo "${{ github.event.pull_request.title }}"
          # Script injection possible!

      # ✅ SAHI — env variable ke through
      - env:
          PR_TITLE: ${{ github.event.pull_request.title }}
        run: echo "$PR_TITLE"
```

> **Key Concepts:** Secrets vs Variables | Environment secrets | Approval gates | GITHUB_TOKEN | Script injection prevention

---

## Q4. 🔵 "Matrix Strategy — Same tests multiple Node versions pe chalao."

### Scenario:
Library banai hai jo Node 16, 18, 20 pe support karna hai — saath mein Ubuntu aur macOS pe bhi.

### Matrix Build:

```yaml
name: Matrix CI

on: [push, pull_request]

jobs:
  test:
    name: Test (Node ${{ matrix.node-version }} on ${{ matrix.os }})
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, macos-latest]
      fail-fast: false    # ek fail ho to baaki mat roko

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm test
```

### Advanced Matrix — Include/Exclude:

```yaml
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']
        os: [ubuntu-latest, windows-latest]

        # Specific combinations add karo
        include:
          - python-version: '3.12'
            os: ubuntu-latest
            experimental: true    # extra variable

        # Specific combinations hatao
        exclude:
          - python-version: '3.9'
            os: windows-latest    # 3.9 + Windows combination skip karo

    steps:
      - name: Experimental step
        if: ${{ matrix.experimental }}   # sirf experimental matrix pe
        run: echo "This is experimental"
```

### Matrix Output Use karna:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          # Dynamic matrix (API se ya file se)
          echo 'matrix={"include":[{"env":"staging"},{"env":"prod"}]}' >> $GITHUB_OUTPUT

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
    steps:
      - run: echo "Deploying to ${{ matrix.env }}"
```

> **Key Concepts:** strategy.matrix | fail-fast | include/exclude | Dynamic matrix | fromJson | Matrix variables

---

## Q5. 🟣 "Reusable Workflow banao — DRY principle follow karo."

### Scenario:
10 alag microservices hain — sab mein same Docker build + ECR push steps copy-paste hain. Centralize karo.

### Reusable Workflow (caller ko doge):

```yaml
# .github/workflows/docker-build-push.yml (REUSABLE)

name: Docker Build and Push (Reusable)

on:
  workflow_call:           # Yahi isko reusable banata hai
    inputs:
      image-name:
        required: true
        type: string
      dockerfile-path:
        required: false
        type: string
        default: './Dockerfile'
      aws-region:
        required: false
        type: string
        default: 'ap-south-1'
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true
    outputs:
      image-tag:
        description: "Built image tag"
        value: ${{ jobs.build.outputs.image-tag }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ inputs.aws-region }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile-path }}
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ inputs.image-name }}:latest
            ${{ steps.login-ecr.outputs.registry }}/${{ inputs.image-name }}:${{ github.sha }}
```

### Caller Workflow (microservice mein):

```yaml
# service-a/.github/workflows/ci.yml

name: Service A CI/CD

on:
  push:
    branches: [main]

jobs:
  # Reusable workflow call karo
  build-image:
    uses: my-org/shared-workflows/.github/workflows/docker-build-push.yml@main
    with:
      image-name: service-a
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  # Output use karo
  deploy:
    needs: build-image
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Image built: ${{ needs.build-image.outputs.image-tag }}"
          # Deploy karo...
```

### Composite Actions (lightweight alternative):

```yaml
# .github/actions/setup-aws/action.yml
name: 'Setup AWS'
description: 'Configure AWS credentials and login to ECR'
inputs:
  aws-region:
    default: 'ap-south-1'
runs:
  using: 'composite'
  steps:
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ inputs.aws-region }}

# Use karo:
- uses: ./.github/actions/setup-aws
  with:
    aws-region: ap-south-1
```

> **Key Concepts:** workflow_call | inputs/outputs/secrets | Reusable workflows | Composite actions | DRY principle

---

## Q6. 🟤 "Complete CI/CD Pipeline — Build → Test → Scan → Push → Deploy to EKS."

### Scenario:
Production-grade pipeline banao jo code push se lekar EKS deployment tak poora automate kare.

### Full Pipeline:

```yaml
name: Production CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-app
  EKS_CLUSTER: my-cluster

jobs:
  # JOB 1: Test
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm
      - run: npm ci
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # JOB 2: Security Scan (code level)
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # SAST — code vulnerabilities
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
      # Dependency vulnerabilities
      - name: npm audit
        run: npm audit --audit-level=high

  # JOB 3: Build & Push Image
  build-push:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: [test, security-scan]    # dono pass hone ke baad
    if: github.ref == 'refs/heads/main'   # sirf main branch pe push pe
    outputs:
      image-tag: ${{ github.sha }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Image
        env:
          IMAGE_URI: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
        run: |
          docker build -t $IMAGE_URI:${{ github.sha }} .
          docker tag $IMAGE_URI:${{ github.sha }} $IMAGE_URI:latest

      # Container security scan
      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          exit-code: '1'
          severity: 'HIGH,CRITICAL'

      - name: Push to ECR
        env:
          IMAGE_URI: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
        run: |
          docker push $IMAGE_URI:${{ github.sha }}
          docker push $IMAGE_URI:latest

  # JOB 4: Deploy to Staging
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: build-push
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${{ env.AWS_REGION }} \
            --name ${{ env.EKS_CLUSTER }}

      - name: Deploy to EKS
        env:
          IMAGE_TAG: ${{ needs.build-push.outputs.image-tag }}
        run: |
          kubectl set image deployment/my-app \
            my-app=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG \
            -n staging
          kubectl rollout status deployment/my-app -n staging --timeout=5m

  # JOB 5: Deploy to Production (approval required)
  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production     # GitHub Settings mein required reviewers set karo
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4
      # ... same deploy steps with -n production
```

> **Key Concepts:** Job dependencies | if conditions | Environment gates | Trivy | EKS deployment | Rollout status

---

## Q7. 🔶 "Workflow fail ho gaya — Debug karo, notifications bhejo, aur rollback karo."

### Scenario:
Production deployment fail ho gayi — team ko pata lagana chahiye aur automatic rollback hona chahiye.

### Error Handling & Notifications:

```yaml
name: Deploy with Rollback

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Get current version (rollback ke liye save karo)
        id: current-version
        run: |
          CURRENT=$(kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "image=$CURRENT" >> $GITHUB_OUTPUT

      - name: Deploy
        id: deploy
        run: |
          kubectl set image deployment/my-app my-app=$NEW_IMAGE
          kubectl rollout status deployment/my-app --timeout=5m

      # Step conditions
      - name: Rollback on failure
        if: failure() && steps.deploy.conclusion == 'failure'
        run: |
          echo "Deploy failed! Rolling back to ${{ steps.current-version.outputs.image }}"
          kubectl rollout undo deployment/my-app
          kubectl rollout status deployment/my-app --timeout=3m

      # Always run — cleanup
      - name: Cleanup
        if: always()    # success aur failure dono pe
        run: docker system prune -f

      # Sirf success pe
      - name: Tag release
        if: success()
        run: git tag v${{ github.run_number }}
```

### Slack/Email Notifications:

```yaml
      # Slack notification (success ya failure)
      - name: Slack — Success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deploy successful!\nRepo: ${{ github.repository }}\nBy: ${{ github.actor }}\nCommit: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Slack — Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Deploy FAILED!\nRepo: ${{ github.repository }}\nBranch: ${{ github.ref }}\nLogs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Step Conditions Cheatsheet:

```yaml
if: success()           # pichla step pass hua
if: failure()           # koi step fail hua
if: always()            # hamesha chalao
if: cancelled()         # workflow cancel hua
if: !cancelled()        # cancel nahi hua

# Specific step check
if: steps.my-step.conclusion == 'failure'
if: steps.my-step.outputs.result == 'changed'

# Branch check
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request'

# Combined
if: failure() && github.ref == 'refs/heads/main'
```

### Debug Mode:

```bash
# Repository mein secret add karo:
ACTIONS_STEP_DEBUG = true
ACTIONS_RUNNER_DEBUG = true

# Ya specific run pe enable karo:
# "Re-run jobs" → "Enable debug logging" checkbox
```

```yaml
      # Debug — context dump karo
      - name: Debug context
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"
```

> **Key Concepts:** if conditions | always/failure/success | Rollback | kubectl rollout undo | Slack notifications | Debug mode

---

## 📌 GitHub Actions — Quick Reference Map

```
Workflow File Structure
├── name
├── on (triggers)
│   ├── push / pull_request
│   ├── schedule (cron)
│   ├── workflow_dispatch (manual)
│   └── workflow_call (reusable)
├── env (global variables)
└── jobs
    ├── job-name
    │   ├── runs-on
    │   ├── environment
    │   ├── needs (dependencies)
    │   ├── if (condition)
    │   ├── strategy.matrix
    │   ├── outputs
    │   └── steps
    │       ├── uses (action)
    │       └── run (command)

Important Contexts
├── ${{ github.sha }}        commit hash
├── ${{ github.ref }}        branch/tag ref
├── ${{ github.actor }}      triggering user
├── ${{ github.run_id }}     unique run ID
├── ${{ secrets.NAME }}      secret value
├── ${{ vars.NAME }}         variable value
└── ${{ needs.job.outputs.key }}  job output

Useful Actions
├── actions/checkout@v4
├── actions/setup-node@v4
├── actions/cache@v4
├── actions/upload-artifact@v4
├── actions/download-artifact@v4
├── aws-actions/configure-aws-credentials@v4
├── aws-actions/amazon-ecr-login@v2
├── docker/build-push-action@v5
└── aquasecurity/trivy-action@master
```

---

*GitHub Actions Interview Prep v1.0 | Concepts: Triggers → Jobs → Secrets → Matrix → Reusable → CD → Debug*
