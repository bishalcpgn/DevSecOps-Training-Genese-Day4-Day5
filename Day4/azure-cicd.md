# Part 1: Azure DevOps CI/CD Pipeline — Fundamentals and Deployment



---

## Table of Contents

1. Introduction to CI/CD and Azure DevOps
2. YAML Pipeline Structure and Keywords
3. Setting Up an Azure DevOps Project
4. Building the Pipeline — Step by Step
5. Environments and Approvals
6. Service Connections
7. Pipeline Triggers, Variables, and Conditions
8. Best Practices
9. Troubleshooting Common Issues
10. Deployment: Node.js to EC2 via rsync

---

## 1. Introduction to CI/CD and Azure DevOps

### 1.1 What is CI/CD?

**Continuous Integration (CI)** is the practice of automatically building every code change the moment a developer commits it. Instead of discovering integration problems days or weeks later, CI catches them within minutes.

**Continuous Delivery (CD)** extends CI by automatically packaging every validated build and making it ready for deployment. Every build that passes quality checks becomes a releasable artifact.

**Continuous Deployment** goes one step further — it automatically deploys every validated build to a target environment without manual intervention.

In the DevSecOps world (covered in Part 2 of this training), "validated" means more than just a successful build. It includes security scans, quality gates, vulnerability checks, and approval gates for production. Part 1 builds the CI/CD foundation that Part 2 adds security on top of.

### 1.2 Why Azure Pipelines?

Azure Pipelines is Microsoft's CI/CD engine inside Azure DevOps. It is commonly chosen when:

- The team already uses Azure DevOps for source control, work items, or test management
- Enterprise features are needed — audit logs, approval workflows, access controls, compliance reporting
- The organization operates in a Microsoft-centric environment

Azure Pipelines can deploy **anywhere** — Azure, AWS, GCP, on-prem servers, Kubernetes, Docker registries. Pipelines run on Microsoft-hosted agents (cloud VMs spun up on demand) or self-hosted agents (machines you manage).

### 1.3 Two pipeline styles

Azure Pipelines supports two authoring approaches:

| | YAML Pipelines | Classic Pipelines |
|---|---|---|
| **Definition lives in** | Repository (`azure-pipelines.yml`) | Azure DevOps database (opaque JSON) |
| **Version controlled** | Yes — alongside the code | No |
| **PR reviewable** | Yes | No |
| **Microsoft investment** | Active development | Maintenance mode only |
| **Recommended** | Yes | No |

This training uses YAML exclusively. 

---

## 2. YAML Pipeline Structure and Keywords

### 2.1 The Hierarchy

A pipeline is a single YAML file structured in layers. Each layer nests inside the one above. Indentation expresses this nesting — there are no braces or end-tags.

```
Pipeline (azure-pipelines.yml)
│
├── Stage: Build
│   └── Job: BuildJob  [runs on ubuntu-latest]
│       ├── Step: Install Node.js
│       ├── Step: npm ci
│       ├── Step: npm run build
│       ├── Step: Archive files
│       └── Step: Publish artifact
│
└── Stage: Deploy
    └── Deployment Job: DeployToEC2  [environment: production]
        ├── Step: Download artifact
        ├── Step: Download SSH key
        ├── Step: rsync files to EC2
        └── Step: Restart app via PM2
```

One pipeline file → stages → jobs → steps. When indentation appears in the YAML throughout this document, this hierarchy is what it expresses.

### 2.2 Keyword Reference Table

| Keyword | Level | What it does |
|---|---|---|
| `trigger` | Top | Branches whose pushes automatically start the pipeline |
| `pr` | Top | Branches whose PRs trigger validation runs |
| `schedules` | Top | Cron-based scheduled runs |
| `variables` | Top or nested | Named values referenced as `$(variableName)` |
| `pool` | Top or job | Which agent VM runs the work |
| `stages` | Top | Container for one or more stages |
| `stage` | Inside `stages` | A phase of the pipeline (Build, Deploy) |
| `jobs` | Inside `stage` | Container for one or more jobs |
| `job` | Inside `jobs` | Unit of work on one agent. Steps share its filesystem |
| `deployment` | Inside `jobs` | Specialized job targeting an environment |
| `steps` | Inside `job` | Ordered list of individual actions |
| `task` | A step type | Pre-built action with inputs e.g. `UseNode@1` |
| `script` | A step type | Inline shell command (bash on Linux) |
| `displayName` | Any level | Human-readable label in the Azure DevOps UI |
| `dependsOn` | Stage or job | Forces execution order |
| `condition` | Stage, job, step | Boolean expression — item runs only if true |
| `continueOnError` | Step | Keep pipeline going even if this step fails |
| `env` | Step | Environment variables scoped to one step |
| `timeoutInMinutes` | Job | Cancel job if it exceeds this duration |

### 2.3 Each Keyword Explained

---

**`trigger`** — *fires on push*

Defines which branches automatically start the pipeline when code is pushed.

```yaml
trigger:
  branches:
    include:
      - main
      - develop
    exclude:
      - feature/*   # pushes to feature/* do NOT trigger this pipeline
```

Without a `trigger` block, the pipeline only runs when manually started.

---

**`pr`** — *fires on pull requests*

Runs the pipeline as PR validation when a pull request is opened or updated targeting the listed branches.

```yaml
pr:
  branches:
    include:
      - main
```

---

**`variables`** — *define once, use everywhere*

```yaml
variables:
  nodeVersion: '20.x'
  ec2Host: '54.123.45.67'
  appName: 'training-app'
```

Referenced anywhere in the pipeline as `$(nodeVersion)`, `$(ec2Host)`, etc. Variables can be defined at pipeline, stage, or job level — a narrower scope overrides a wider one.

**Secret variables** are marked with the lock icon in Azure DevOps. They never appear in pipeline logs — Azure DevOps masks them automatically.

---

**`pool`** — *where the pipeline runs*

```yaml
pool:
  vmImage: 'ubuntu-latest'
```

Common values: `ubuntu-latest`, `windows-latest`, `macOS-latest`. A self-hosted agent pool is referenced by name instead of `vmImage`.

---

**`stages`** — *top-level phases*

Stages run **sequentially by default**. Each stage waits for the previous to finish before starting.

```yaml
stages:
  - stage: Build
    ...
  - stage: Deploy
    dependsOn: Build
    ...
```

---

**`jobs`** — *units of work within a stage*

Jobs within the same stage run **in parallel by default.** This is a common point of confusion — stages are sequential, but jobs within a stage are parallel unless `dependsOn` is used on the job.

---

**`job`** — *runs on one agent*

A job starts, checks out the repo on a fresh agent VM, runs all its steps top-to-bottom, and finishes. The VM is discarded after. Files do not persist to the next job unless published as artifacts first.

```yaml
- job: BuildJob
  displayName: 'Build Node.js app'
  pool:
    vmImage: 'ubuntu-latest'
  timeoutInMinutes: 30
  steps:
    - script: npm ci
```

---

**`deployment`** — *a job with environment awareness*

Use `deployment` instead of `job` when the step actually deploys to a target. It integrates with the Environments feature — approval gates, deployment history, branch controls.

```yaml
- deployment: DeployToEC2
  environment: 'production'
  strategy:
    runOnce:
      deploy:
        steps:
          - script: echo "deploying..."
```

---

**`task`** — *pre-built reusable action*

Tasks are identified by `Name@MajorVersion`. Pinning the major version means bug fixes arrive automatically, but breaking changes do not.

```yaml
- task: UseNode@1
  inputs:
    version: '20.x'
  displayName: 'Install Node.js'
```

> **Why `@1`?** The number is the major version. Always pin it. `@latest` can introduce breaking changes overnight.

---

**`script`** — *inline shell command*

Shorthand for running a bash command on Linux/macOS agents.

```yaml
- script: npm ci
  displayName: 'Install dependencies'
```

Multi-line with `|` (block-literal syntax):

```yaml
- script: |
    npm ci
    npm run build
  displayName: 'Install and build'
```

---

**`condition`** — *run only when...*

```yaml
# Only deploy from main branch
condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

Common built-in functions: `succeeded()`, `failed()`, `always()`, `succeededOrFailed()`.

---

**`env`** — *environment variables for one step*

Passes secrets to a step without exposing them globally.

```yaml
- script: curl -u $(SONAR_TOKEN): https://sonarcloud.io/api/...
  env:
    SONAR_TOKEN: $(sonarToken)
```

---

### 2.4 Indentation Rules

YAML uses **spaces, not tabs**, for indentation. Two spaces per level is the convention. One misaligned space breaks the entire file.

```yaml
stages:               # level 0
  - stage: Build      # level 1 — 2 spaces
    jobs:             # level 2 — 4 spaces
      - job: Build    # level 3 — 6 spaces
        steps:        # level 4 — 8 spaces
          - script: npm ci   # level 5 — 10 spaces
```

Use VS Code with the YAML extension — it highlights indentation errors before committing.

### 2.5 The Smallest Legal Pipeline

Most keywords are optional. This is a complete, valid Azure DevOps pipeline:

```yaml
steps:
  - script: echo "Hello, Azure Pipelines!"
```

Every keyword added beyond this is structure added to handle a real-world need — multiple environments, conditional deploys, security gates. Start simple. Add structure when the need is clear.

### 2.6 Pipeline Execution Flow

How a pipeline moves from code push to deployed application:

```
Developer pushes to main
         |
         v
[Azure DevOps detects trigger]
         |
         v
+---------------------------+
|     STAGE 1: Build        |
|  (fresh ubuntu-latest VM) |
|                           |
|  1. Install Node.js       |
|  2. npm ci                |
|  3. npm run build         |
|  4. Archive to .zip       |
|  5. Publish artifact      |
+---------------------------+
         |
         | artifact stored in Azure DevOps
         |
         v
+---------------------------+
|    STAGE 2: Deploy        |
|  (different fresh VM)     |
|                           |
|  1. Download artifact     |
|  2. Download SSH key      |
|  3. rsync to EC2          |
|  4. npm ci --production   |
|  5. pm2 restart           |
+---------------------------+
         |
         | App live on EC2
         |
         v
+---------------------------+
|   STAGE 3: ZAP DAST Scan  |
|  (another fresh VM        |
|   with Docker)            |
|                           |
|  1. Wait for app (HTTP)   |
|  2. Prepare report dir    |
|  3. Run ZAP baseline      |
|     scan via Docker       |
|  4. Publish zap-reports   |
|     artifact (HTML/JSON)  |
+---------------------------+
         |
         v
  Reports available in
  Azure DevOps artifacts
```

**Key point:** Build, Deploy, and ZAP Scan each run on completely separate VMs. Files do not carry over automatically — artifacts are the bridge between stages, and the live application URL is the bridge from Deploy to the ZAP scanner.

---

## 3. Setting Up an Azure DevOps Project

### 3.1 Creating an organization

Navigate to **https://dev.azure.com** and sign in with a Microsoft account.

> **2026 note:** Creating a new Azure DevOps organization now requires an active Azure pay-as-you-go subscription. Organizations created before July 2025 are grandfathered and do not require a subscription. For this training, an existing organization is used.

### 3.2 Creating a project

1. On the organization dashboard, click **+ New project** (top right)
2. **Project name:** `devsecops-lab`
3. **Visibility:** Private
4. **Version control:** Git
5. **Work item process:** Agile
6. Click **Create**

The project overview page shows Repos, Pipelines, Boards, and other services in the left navigation.

### 3.3 Importing the starter repository

The training uses a pre-built Node.js application from GitHub. Import it into Azure Repos:

1. Click **Repos** in the left nav
2. Click **Import repository**
3. **Repository type:** Git
4. **Clone URL:** `https://github.com/bishalcpgn/DevSecOps-Training-Genese.git`
5. Click **Import**

After import, browse `package.json` and review the `scripts` section — the `build` and `start` scripts are what the pipeline will invoke.

---

## 4. Building the Pipeline — Step by Step

Rather than presenting a complete pipeline and explaining it afterward, this section builds it in two incremental steps. Each step introduces new concepts.

### 4.1 Create the pipeline

1. Click **Pipelines** in the left nav → **Create Pipeline**
2. **Where is your code?** → Azure Repos Git
3. Select the repository 
4. **Configure your pipeline** → Starter pipeline
5. The YAML editor opens — replace its contents completely with each step below

### 4.2 Step 1 — Build stage with artifact publishing

The pipeline installs dependencies, builds the application, archives the output, and publishes it as a pipeline artifact for the Deploy stage to consume. The `stages:` / `stage:` / `jobs:` / `job:` hierarchy scales cleanly as more stages are added.

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  nodeVersion: '20.x'

stages:

- stage: Build
  displayName: 'Build'
  jobs:
  - job: BuildJob
    displayName: 'Build Node.js application'
    pool:
      vmImage: 'ubuntu-latest'
    steps:

    - task: UseNode@1
      inputs:
        version: $(nodeVersion)
      displayName: 'Install Node.js $(nodeVersion)'

    - script: npm install --legacy-peer-deps
      displayName: 'Install dependencies'

    - script: npm run build --if-present
      displayName: 'Build application'

    - task: ArchiveFiles@2
      displayName: 'Archive build output'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
        includeRootFolder: false
        archiveType: 'zip'
        archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
        replaceExistingArchive: true

    - task: PublishPipelineArtifact@1
      displayName: 'Publish artifact: drop'
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: 'drop'
        publishLocation: 'pipeline'
```

**Concepts introduced:**

- `pr:` — pipeline now runs as PR validation in addition to push
- `stages:` / `stage:` / `jobs:` / `job:` — structured hierarchy
- `npm ci` — clean install of exact dependency versions from `package-lock.json`
- `npm run build --if-present` — runs only if a `build` script exists in `package.json`
- `ArchiveFiles@2` — zips the build output so the Deploy stage can rsync it to EC2
- `PublishPipelineArtifact@1` — uploads the zip to Azure DevOps storage

**Verify:**
- Build stage shows green
- Run summary → **Artifacts** → `drop` contains the `.zip` file

---

### 4.3 Why artifacts matter

Each stage runs on a **different, ephemeral agent VM**. When the Build stage finishes, that VM is discarded. The Deploy stage gets a completely new VM with an empty filesystem.

```
+-----------------------------+          +-----------------------------+
|   BUILD AGENT (VM #1)       |          |   DEPLOY AGENT (VM #2)      |
|                             |          |                             |
|  npm ci                     |          |  (empty filesystem)         |
|  npm run build              |          |                             |
|  zip output                 |          |  DownloadPipelineArtifact   |
|  PublishPipelineArtifact -->|--------->|  --> drop/build.zip         |
|                             |  stored  |                             |
|  (VM discarded)             |  in ADO  |  rsync to EC2               |
+-----------------------------+          +-----------------------------+
```

The artifact named `drop` is the bridge. Build publishes it to Azure DevOps storage. Deploy downloads it automatically. Without `PublishPipelineArtifact@1`, the Deploy stage has nothing to work with.

---

### 4.4 Step 2 — Conditions and PR gates

Now that `pr:` is in the YAML, the pipeline runs on both pushes to `main` and on PRs. But a PR should only validate — never deploy.

The `condition:` keyword controls this:

```yaml
- stage: Deploy
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

This reads: run only if Build succeeded AND the source branch is `main`.

When a PR runs, `Build.SourceBranch` is `refs/pull/N/merge` — not `refs/heads/main` — so the Deploy stage is automatically skipped. No special configuration needed; it is a natural consequence of the condition evaluating to false.

---

### 4.5 Step 3 — The complete pipeline

The full pipeline with Build and Deploy stages. The Deploy stage is covered in detail in Section 10.

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  nodeVersion: '20.x'
  ec2Host: '13.206.194.54'          # EC2 public IP (Don't Hardcode)
  ec2User: 'ubuntu'
  deployPath: '/var/www/training-app'
  appName: 'training-app'
  appUrl: 'http://13.206.194.54:3000'

stages:

# ============================================================
# STAGE 1 — Build
# ============================================================
- stage: Build
  displayName: 'Build'
  jobs:
  - job: BuildJob
    pool:
      vmImage: 'ubuntu-latest'
    steps:

    - task: UseNode@1
      inputs:
        version: $(nodeVersion)
      displayName: 'Install Node.js $(nodeVersion)'

    - script: npm install --legacy-peer-deps
      displayName: 'Install dependencies'

    - script: npm run build
      displayName: 'Build Vite application'

    # Archive ONLY the dist folder (pre-built static SPA assets)
    - task: ArchiveFiles@2
      displayName: 'Archive dist output'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)/dist'
        includeRootFolder: false
        archiveType: 'zip'
        archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
        replaceExistingArchive: true

    - task: PublishPipelineArtifact@1
      displayName: 'Publish artifact: drop'
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: 'drop'

# ============================================================
# STAGE 2 — Deploy to EC2
# Only runs on pushes to main, not on PRs
#
# Changed from deployment job to normal job
# to avoid Azure DevOps environment approval checks.
#
# This deploys a Vite application using PM2 static hosting.
# ============================================================

- stage: Deploy
  displayName: 'Deploy to EC2'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

  jobs:
  - job: DeployToEC2
    displayName: 'Deploy Vite app to EC2'

    pool:
      vmImage: 'ubuntu-latest'

    steps:

    # ------------------------------------------------------------
    # Download build artifact
    # ------------------------------------------------------------
    - task: DownloadPipelineArtifact@2
      displayName: 'Download build artifact'
      inputs:
        artifact: 'drop'
        path: '$(Pipeline.Workspace)/drop'

    # ------------------------------------------------------------
    # Download SSH private key
    # ------------------------------------------------------------
    - task: DownloadSecureFile@1
      name: sshKey
      displayName: 'Download SSH private key'
      inputs:
        secureFile: 'devsecops-training-server.pem'

    # ------------------------------------------------------------
    # Unzip dist locally, then sync to EC2
    # ------------------------------------------------------------
    - script: |
        chmod 600 $(sshKey.secureFilePath)

        mkdir -p $(Pipeline.Workspace)/dist

        unzip -q $(Pipeline.Workspace)/drop/$(Build.BuildId).zip \
          -d $(Pipeline.Workspace)/dist

        ssh -i $(sshKey.secureFilePath) \
          -o StrictHostKeyChecking=no \
          $(ec2User)@$(ec2Host) \
          "mkdir -p $(deployPath)"

        rsync -avz --delete \
          -e "ssh -i $(sshKey.secureFilePath) -o StrictHostKeyChecking=no" \
          $(Pipeline.Workspace)/dist/ \
          $(ec2User)@$(ec2Host):$(deployPath)/

        echo "dist/ synced to EC2"

      displayName: 'Sync dist to EC2'

    # ------------------------------------------------------------
    # Restart PM2 on EC2 (serves pre-built dist as static SPA)
    # ------------------------------------------------------------
    - script: |
        chmod 600 $(sshKey.secureFilePath)

        ssh -i $(sshKey.secureFilePath) \
          -o StrictHostKeyChecking=no \
          $(ec2User)@$(ec2Host) << 'REMOTE'
        set -e

        echo "Restarting PM2 application..."
        pm2 delete $(appName) 2>/dev/null || true

        pm2 serve $(deployPath) 3000 \
          --name $(appName) \
          --spa

        pm2 save

        echo "Application started successfully"
        REMOTE

      displayName: 'Restart PM2 on EC2'

# ============================================================
# STAGE 3 — OWASP ZAP DAST Scan
# Runs a baseline scan against the live app on EC2.
# Reports (HTML + JSON + MD) are published as pipeline artifacts.
#
# Baseline scan = passive scan + spider only (safe, ~2-5 min).
# Use full-scan.py later for active scanning once triaged.
# ============================================================

- stage: ZAPScan
  displayName: 'OWASP ZAP DAST Scan'
  dependsOn: Deploy
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

  jobs:
  - job: ZapBaselineScan
    displayName: 'Run ZAP baseline scan'

    pool:
      vmImage: 'ubuntu-latest'

    steps:

    # ------------------------------------------------------------
    # Wait for app to be reachable before scanning
    # Polls the URL every 5s, up to ~2 minutes
    # ------------------------------------------------------------
    - script: |
        echo "Waiting for $(appUrl) to respond..."

        for i in {1..24}; do
          status=$(curl -s -o /dev/null -w "%{http_code}" $(appUrl) || echo "000")
          if [ "$status" = "200" ]; then
            echo "App is live (HTTP $status) after $((i*5))s"
            exit 0
          fi
          echo "Attempt $i: HTTP $status — retrying in 5s..."
          sleep 5
        done

        echo "App did not become reachable within 2 minutes"
        exit 1
      displayName: 'Wait for app to be live'

    # ------------------------------------------------------------
    # Prepare report directory with permissions ZAP container can write to
    # ------------------------------------------------------------
    - script: |
        mkdir -p $(Pipeline.Workspace)/zap-reports
        chmod 777 $(Pipeline.Workspace)/zap-reports
      displayName: 'Prepare ZAP report directory'

    # ------------------------------------------------------------
    # Run ZAP baseline scan via Docker
    # The -I flag prevents non-zero exit on warnings (report-only mode)
    # ------------------------------------------------------------
    - script: |
        docker run --rm \
          -v $(Pipeline.Workspace)/zap-reports:/zap/wrk/:rw \
          ghcr.io/zaproxy/zaproxy:stable \
          zap-baseline.py \
            -t $(appUrl) \
            -r zap-report.html \
            -J zap-report.json \
            -w zap-report.md \
            -I

        echo "ZAP scan completed"
        ls -la $(Pipeline.Workspace)/zap-reports
      displayName: 'Run OWASP ZAP baseline scan'

    # ------------------------------------------------------------
    # Publish ZAP reports as pipeline artifact
    # ------------------------------------------------------------
    - task: PublishPipelineArtifact@1
      displayName: 'Publish ZAP reports'
      condition: always()    # publish even if scan reports warnings
      inputs:
        targetPath: '$(Pipeline.Workspace)/zap-reports'
        artifact: 'zap-reports'
```

**Concepts introduced in the ZAP stage:**

- **DAST (Dynamic Application Security Testing)** — scans the running application from the outside, just like an attacker would. Complements SAST (covered in Part 2), which scans source code.
- **`dependsOn: Deploy`** — ensures the scan only runs after a successful deployment.
- **Health check before scanning** — the `curl` loop polls the app URL until it responds with HTTP 200, with a 2-minute timeout. This is more reliable than a blind `sleep`.
- **ZAP baseline scan** — runs passive scan + spider only. Safe to point at any environment (~2-5 min). The full active scan (`zap-full-scan.py`) is more invasive and reserved for dedicated test environments.
- **Docker on the build agent** — Microsoft-hosted agents ship with Docker preinstalled. ZAP runs in a container, leaving EC2 clean.
- **The `-I` flag** — tells ZAP not to exit with a non-zero code on warnings. First-time scans flag many low-severity findings (missing security headers, info-disclosure cookies) that are normal baseline noise. Remove `-I` later to fail the build on new findings.
- **Three report formats** — HTML (human review), JSON (parsing/dashboards), Markdown (PR comments later).
- **`condition: always()` on publish** — reports are published even if the scan step reports warnings, ensuring findings are never lost.

**Security group requirement:**

The build agent must be able to reach port 3000 on EC2. The security group inbound rule for port 3000 from `0.0.0.0/0` (already in place from Section 10.2) is what makes this work.

**Verify:**
- ZAP stage shows green
- Run summary → **Artifacts** → `zap-reports` contains `zap-report.html`, `zap-report.json`, `zap-report.md`
- Download `zap-report.html` and open in a browser — this is the baseline to triage from

The application is live on http://13.206.194.54:3000/. 

---


## 5. Environments and Approvals

### 5.1 What is an environment?

When `deployment:` specifies `environment: 'production'`, Azure DevOps creates a logical **Environment** object under **Pipelines → Environments**. It appears after the first deploy run.

Environments provide:
- **Deployment history** — every deploy recorded with who triggered it, from which branch, which build
- **Approvals** — a named person must approve before the deployment job runs
- **Checks** — automated checks that must pass before deploy proceeds
- **Branch controls** — restrict the environment to accept deploys from specific branches only

### 5.2 Adding a manual approval gate

1. **Pipelines → Environments → production**
2. Three-dot menu → **Approvals and checks**
3. Click **+** → **Approvals**
4. Add an approver
5. Set timeout — e.g., 24 hours before auto-cancel
6. Save

Every Deploy stage run now pauses before the `DeployToEC2` job. The approver reviews what changed and clicks Approve or Reject.

### 5.3 Branch controls on environments

To restrict an environment to only accept deployments from `main`:

1. **Environments → production → Approvals and checks → + → Branch control**
2. **Allowed branches:** `refs/heads/main`
3. Save

Even a manually triggered pipeline from a different branch will be blocked at the environment gate.

---

## 6. Service Connections

Service connections are how Azure DevOps authenticates to external services — cloud providers, artifact registries, SSH endpoints, SonarCloud, and more.

### 6.1 Types of service connections

| Type | Used for |
|---|---|
| Azure Resource Manager | Deploying to Azure services |
| **SSH** | Connecting to Linux servers — used in this training |
| SonarCloud | SAST in Part 2 |
| Docker Registry | Pushing/pulling Docker images |
| Generic | Any HTTP service with token auth |
| GitHub | Connecting to GitHub repos |

### 6.2 SSH service connection for EC2

Service connections live at **Project Settings → Service connections**.

To create an SSH service connection:

1. **New service connection** → **SSH**
2. Fill in:
   - **Host name:** EC2 public IP
   - **Port:** 22
   - **Username:** `ubuntu` (Amazon Linux uses `ec2-user`)
   - **Private key:** Paste the contents of the `.pem` file
   - **Service connection name:** `EC2-Deploy-Connection`
3. Check **Grant access permission to all pipelines**
4. **Save**

### 6.3 Secure Files — the approach used in this training

The pipeline in Section 4 uses **Secure Files** instead of an SSH service connection. This approach works well when the same key is reused across multiple pipelines:

1. **Pipelines → Library → Secure files**
2. **+ Secure file** → upload the `.pem` key as `devsecops-training-server.pem`
3. The pipeline retrieves it at runtime with `DownloadSecureFile@1`

Secure files are encrypted at rest. They are only accessible to pipelines explicitly granted access.

### 6.4 Why secrets must never go into YAML

A private key committed to a git repository:
- Is visible to anyone with read access to the repo
- Persists in git history even after deletion
- Is exposed in every fork of the repository
- Can be found by automated secret scanners

Azure DevOps Secure Files and secret variables exist specifically to prevent this.

---

## 7. Pipeline Triggers, Variables, and Conditions

### 7.1 Trigger behavior at a glance

```
EVENT                          TRIGGER TYPE        RESULT
------------------------------------------------------------
Push to main              -->  trigger: main    -->  Build + Deploy run
Push to develop           -->  trigger: develop -->  Build only (Deploy condition fails)
Push to feature/x         -->  excluded         -->  Nothing runs
PR opened against main    -->  pr: main         -->  Build only (Deploy condition fails)
Scheduled at 02:00        -->  schedules        -->  Build + Deploy run
```

### 7.2 Variables — types and scopes

| Type | Where defined | Secret-capable | Scope |
|---|---|---|---|
| Inline YAML | In the YAML file | No — visible in source | Pipeline-wide |
| Pipeline UI variables | Pipeline → Edit → Variables | Yes — lock icon | Pipeline-wide |
| Variable groups (Library) | Pipelines → Library | Yes | Shared across pipelines |
| Azure Key Vault linked | Library → Key Vault link | Yes — stored in KV | Shared across pipelines |

**Rule:** Never put secrets in YAML. They become part of git history and are visible to anyone with repo access, permanently.

### 7.3 Conditions

```yaml
# Run only if all previous stages succeeded (default)
condition: succeeded()

# Run even if previous stages failed — useful for cleanup or notifications
condition: always()

# Run only when a previous stage failed
condition: failed()

# Only deploy from main branch
condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# Skip on PR builds entirely
condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
```

### 7.4 Predefined system variables

Azure DevOps injects these into every pipeline run automatically:

| Variable | Value | Example |
|---|---|---|
| `$(Build.BuildId)` | Unique run number | `142` |
| `$(Build.SourceBranch)` | Full ref of triggering branch | `refs/heads/main` |
| `$(Build.SourceBranchName)` | Short branch name | `main` |
| `$(Build.Reason)` | What triggered the run | `IndividualCI`, `PullRequest`, `Schedule` |
| `$(System.DefaultWorkingDirectory)` | Repo checkout path on agent | `/home/runner/work/...` |
| `$(Build.ArtifactStagingDirectory)` | Temp folder for publishing artifacts | `/home/runner/work/_temp/...` |
| `$(Pipeline.Workspace)` | Root workspace folder | `/home/runner/work/1` |
| `$(Agent.OS)` | OS of the agent | `Linux` |

---



## 8. Best Practices

- **YAML over Classic.** Version the pipeline with the code. Roll back pipeline changes via git. Review pipeline changes in pull requests.
- **Pin task versions.** `UseNode@1` is stable and predictable. `UseNode@latest` can introduce breaking changes overnight.
- **Use `npm ci` not `npm install` in CI.** Reproducible installs from `package-lock.json`. Faster and more reliable.
- **Define `timeoutInMinutes` on long-running jobs.** Prevents pipelines hanging indefinitely on stuck processes.
- **Gate deploys with `condition:`.** PRs should build, never deploy.
- **Store all secrets in Variable Groups or Secure Files.** Nothing sensitive in YAML or source code.
- **Use environments for production deploys.** Audit trail, approvals, and branch controls in one place.
- **Always publish artifacts.** They are the rollback mechanism if a deploy needs to be reversed.
- **Add `displayName` to every step.** Pipeline logs become readable. Debugging becomes fast.
- **Separate Build and Deploy stages.** A failed deploy can be retried without rebuilding.
- **Use `continueOnError` deliberately.** Only for steps where failure is genuinely non-blocking.

---

## 9. Troubleshooting Common Issues

| Symptom | Most likely cause | Fix |
|---|---|---|
| "No hosted parallelism has been purchased or granted" | New org without free grant | Submit https://aka.ms/azpipelines-parallelism-request — allow 1-3 business days |
| Pipeline queued 10+ minutes | Only 1 free parallel job, another is running | Wait, or register a self-hosted agent |
| `npm ci` fails with peer-dependency error | `package-lock.json` out of sync | Run `npm install` locally and commit the updated lock file |
| Artifact `drop` not found in Deploy stage | `PublishPipelineArtifact@1` was skipped | Check Build stage logs — the publish step may not have run |
| Deploy stage silently skipped | Condition evaluated to false | Add `- script: echo $(Build.SourceBranch)` before Deploy stage to verify the value |
| SSH connection refused in rsync step | Port 22 blocked in EC2 security group | Add an inbound rule in AWS for port 22 |
| SSH hangs waiting for input | Host fingerprint verification prompt | Always include `-o StrictHostKeyChecking=no` in pipeline SSH commands |
| rsync succeeds but app serves old code | PM2 serving old process | SSH into EC2, run `pm2 list` and `pm2 logs` to diagnose |
| YAML won't save — "unexpected token" | Indentation error — tab instead of space | Check for tabs; use VS Code with the YAML extension |
| `DownloadSecureFile@1` — file not found | Secure file not uploaded or name mismatch | Verify the file exists in Pipelines → Library → Secure files with the exact filename |

---

## 10. Deployment: Node.js to EC2 via rsync

### 10.1 Why EC2 and rsync?

Azure Pipelines can deploy to any server — not only Azure services. The EC2 + rsync pattern is one of the most transferable deployment approaches in the industry. It works on any Linux server: AWS EC2, GCP Compute Engine, DigitalOcean Droplets, bare-metal servers, on-premises VMs.

Understanding this pattern means understanding deployment pipelines that work everywhere, not just within one cloud provider's ecosystem.

### 10.2 EC2 prerequisites

The following is set up on the EC2 instance before the pipeline runs:

```bash
# Node.js 20 LTS via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# PM2 — process manager, keeps Node.js app running after SSH session ends
# PM2 also serves the built dist/ folder as static files (pm2 serve)
sudo npm install -g pm2
pm2 startup systemd   # auto-start PM2 on server reboot

# Application directory
sudo mkdir -p /var/www/training-app
sudo chown ubuntu:ubuntu /var/www/training-app
```

AWS EC2 security group inbound rules required:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | SSH | 0.0.0.0/0 | Azure DevOps agent SSH access |
| 3000 | TCP | 0.0.0.0/0 | Application access (served by PM2) |

### 10.3 What rsync does

`rsync` is a file synchronization tool that:
- Transfers only what changed — subsequent deploys send only modified files
- Deletes files on the target that no longer exist locally (with `--delete`)
- Preserves permissions and timestamps
- Works over SSH — same transport as a regular SSH connection

`scp` copies everything every time. rsync is the production-appropriate habit.

### 10.4 The Deploy stage — step by step

**Step 1: Download the build artifact**

```yaml
- task: DownloadPipelineArtifact@2
  inputs:
    artifact: 'drop'
    path: '$(Pipeline.Workspace)/drop'
```

Downloads the `.zip` published by the Build stage. The Deploy agent's fresh VM has nothing until this step runs.

**Step 2: Download the SSH private key**

```yaml
- task: DownloadSecureFile@1
  name: sshKey
  inputs:
    secureFile: 'ec2-deploy-key.pem'
```

Retrieves the encrypted `.pem` from Azure DevOps Secure Files. The local path is accessible as `$(sshKey.secureFilePath)` in subsequent steps.

**Step 3: Set key permissions and rsync**

```bash
chmod 600 $(sshKey.secureFilePath)
```

SSH requires the private key to have `600` permissions (owner read/write only). Without this, SSH refuses to use the key with a `bad permissions` error.

```bash
rsync -avz --delete \
  -e "ssh -i $(sshKey.secureFilePath) -o StrictHostKeyChecking=no" \
  $(Pipeline.Workspace)/app/ \
  $(ec2User)@$(ec2Host):$(deployPath)/
```

| Flag | Meaning |
|---|---|
| `-a` | Archive mode — preserves permissions, timestamps, symlinks |
| `-v` | Verbose — shows each file being transferred |
| `-z` | Compress in transit — faster on slower connections |
| `--delete` | Delete EC2 files no longer present locally |
| `-e "ssh ..."` | Use SSH as transport with the specified key |
| `StrictHostKeyChecking=no` | Skip host fingerprint prompt |

**Step 4: Install production dependencies and restart app**

```bash
cd $(deployPath)
npm ci --production
pm2 restart training-app 2>/dev/null || pm2 start app.js --name training-app
pm2 save
```

- `npm ci --production` — installs only production dependencies. Dev tools are excluded from the server.
- `pm2 restart ... || pm2 start ...` — restarts the existing process if it exists; starts a new one if not.
- `pm2 save` — persists the process list so it survives a server reboot.

### 10.5 What the pipeline logs show during a successful deployment

```
Step: Download build artifact
  Downloading artifact 'drop' to /home/runner/work/1/drop/

Step: Download SSH private key
  Downloaded secure file to /tmp/ec2-deploy-key.pem

Step: rsync files to EC2
  sending incremental file list
  ./
  app.js
  package.json
  package-lock.json
  src/
  src/index.js
  sent 14,231 bytes  received 312 bytes
  Files synced to EC2

Step: Install dependencies and restart app
  added 47 packages in 3.2s
  [PM2] Restarting 'training-app'
  [PM2] 'training-app' restarted
  Application restarted
```

After the pipeline completes, the application is accessible at the EC2 public IP on port 3000.

### 10.6 Making this production-ready — discussion points

The pipeline as shown is functional for a training environment. A production version would add:

**Health check after restart**
```bash
sleep 5
curl -f http://$(ec2Host):3000/ || \
  (echo "Health check failed"; pm2 logs --nostream; exit 1)
```

**Rollback mechanism** — keep previous build artifacts. If a deploy fails, trigger the pipeline with the previous `BuildId` to re-deploy a known-good version.

**Zero-downtime restart with PM2**
```bash
pm2 reload $(appName)   # reload instead of restart — zero-downtime on multi-process apps
```

**Secrets in variable groups** — the `ec2Host` variable is currently in the YAML and visible in source. Move it to a variable group in the Library for proper secrets management.

**Multiple environments** — add a `Staging` environment between Build and Deploy. Deploy to staging first, verify, then deploy to production with an approval gate.

### 10.7 What Part 2 adds to this pipeline

In Part 2, a `SecurityScan` stage is inserted between Build and Deploy:

```
Build  -->  SecurityScan  -->  Deploy  -->  ZAP
           (npm audit +
            Dep-Check +
            SonarCloud)
```

If SonarCloud's quality gate fails, if npm audit finds a critical vulnerability, or if Dependency-Check reports a high-severity CVE — the Deploy stage never runs. Security gates become pipeline gates. This is what makes the pipeline DevSecOps rather than just CI/CD.

---

## Preparing for Part 2

Before Part 2 (DevSecOps Security Integration), the following is needed:

**SonarCloud account**
Create at https://sonarcloud.io .

**NVD API key**
Required for OWASP Dependency-Check to query the vulnerability database at reasonable speed.
Request for free at https://nvd.nist.gov/developers/request-an-api-key — arrives by email within minutes.
Store it as a secret variable named `NVD_API_KEY` in the Azure DevOps pipeline.

Part 2 builds directly on the pipeline from Section 4.6. The YAML is the starting point — security scan stages are inserted between Build and Deploy.

---

*Document version: 2026.05 | Part 1 of 2 |  DevSecOps Training Series*
