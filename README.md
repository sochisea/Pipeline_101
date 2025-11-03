## 🌱 Overview

A **Jenkins pipeline** is just a script (usually named `Jenkinsfile`) that tells Jenkins:

> “Here’s what to do, and in what order.”

Think of it like a *recipe* for automation — Jenkins is the cook, and your Jenkinsfile is the recipe.

In your case, it runs **three stages** (Init → Build → Test).
Each stage contains **steps**, which are individual shell commands or Jenkins functions.

---

## 🔍 Step-by-step Explanation

### 1️⃣ Pipeline & agent

```groovy
pipeline {
  agent any
```

* `pipeline { ... }`
  means we’re writing a **Declarative Pipeline** (the modern, simpler syntax).

* `agent any`
  tells Jenkins to run this pipeline on *any available machine (agent)*.
  If you had multiple build nodes (e.g., Linux, Windows, Docker), you could pick one.

📘 *In Jenkinsfile Runner (GitHub Actions), it just runs on the Ubuntu container.*

---

### 2️⃣ Options block

```groovy
options {
  skipDefaultCheckout(true)
  catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS')
}
```

* `skipDefaultCheckout(true)`
  Jenkins usually checks out your Git repo automatically before running.
  But since **GitHub Actions already did the checkout**, we skip it — this avoids “UNSTABLE” warnings.

* `catchError(...)`
  This tells Jenkins:

  > “Even if something marks the build as UNSTABLE, treat it as SUCCESS.”

  It’s a safeguard so your training pipelines finish green.

---

### 3️⃣ Parameters

```groovy
parameters {
  string(name: 'NAME', defaultValue: 'World', description: 'Who to greet')
  booleanParam(name: 'DO_BUILD', defaultValue: true, description: 'Run Build stage?')
}
```

* Jenkins allows you to pass *parameters* when you start a build.
* Here you define:

  * `NAME` → a text input (default “World”)
  * `DO_BUILD` → a true/false checkbox
* Inside the script, you access them with `params.NAME` and `params.DO_BUILD`.

🧠 Example:
If you start a run and type “Eliyahu”, the Init stage will say:

> “Hi Eliyahu! APP_ENV=dev”

---

### 4️⃣ Environment

```groovy
environment { APP_ENV = 'dev' }
```

* Global environment variables live here.
* You can use them in any stage via `${env.APP_ENV}`.
* In DevOps, this is how you separate dev / test / prod configurations.

---

### 5️⃣ Stages — the core flow

Stages are like **chapters** in your automation story.

#### 🧩 Stage 1 — Init

```groovy
stage('Init') {
  steps {
    echo "Hi ${params.NAME}! APP_ENV=${env.APP_ENV}"
    sh '''
      set -eu
      mkdir -p build reports/junit
      echo "Init OK" > build/init.txt
    '''
  }
}
```

* `stage('Init')` defines a named section of the pipeline.
* `echo` prints messages to Jenkins logs.
* `sh` runs shell commands (Linux terminal inside the runner).

🧠 It creates folders and a file `build/init.txt` — just proving things run.

---

#### 🧱 Stage 2 — Build

```groovy
stage('Build') {
  when { expression { return params.DO_BUILD } }
  steps {
    timeout(time: 1, unit: 'MINUTES') {
      retry(1) {
        sh '''
          set -eu
          echo "Building for ${APP_ENV}" > build/build.txt
          echo "Build OK"
        '''
      }
    }
  }
}
```

* `when { ... }` checks a condition: run only if `DO_BUILD == true`.
  (If false, Jenkins skips this stage.)

* `timeout(1 minute)` prevents a stage from hanging forever.

* `retry(1)` means “if it fails once, try again once.”

* Inside, shell commands simulate building your app.

🧱 Output → file `build/build.txt` with the text “Build OK”.

---

#### 🧪 Stage 3 — Test

```groovy
stage('Test') {
  steps {
    sh '''
      set -eu
      cat > reports/junit/demo.xml <<'XML'
      <?xml version="1.0" encoding="UTF-8"?>
      <testsuite name="demo" tests="2" failures="0" errors="0" skipped="0" time="0.01">
        <testcase classname="calc.Add" name="adds" time="0.001"/>
        <testcase classname="calc.Mul" name="multiplies" time="0.001"/>
      </testsuite>
      XML
    '''
    junit testResults: 'reports/junit/*.xml', allowEmptyResults: true
  }
}
```

* Creates a fake **JUnit XML test report** (standard format for test results).
* `junit` step tells Jenkins:

  > “Read test results from these files and show them in the UI.”

✅ So you learn how Jenkins handles test reporting.

---

### 6️⃣ Post actions

```groovy
post {
  always {
    echo 'Level 1 finished.'
  }
}
```

* The `post` block runs *after* all stages — no matter success or failure.
* Common uses:

  * Send Slack/email notifications
  * Clean up workspaces
  * Archive artifacts

---

## 🔁 Full logic summary (like a flowchart)

```
Start
 ├─ Stage 1: Init  → create folders, greet user
 ├─ If DO_BUILD=true → Stage 2: Build  → make build.txt
 ├─ Stage 3: Test   → create fake junit file
 └─ Post: always print “Level 1 finished.”
Done ✅
```

---

## 🧩 Concepts you just learned

| Concept                 | Description                                   |
| ----------------------- | --------------------------------------------- |
| **Pipeline**            | Main block defining all stages                |
| **Agent**               | Machine/environment that runs the steps       |
| **Stage**               | Logical step in your process                  |
| **Step**                | Actual command or Jenkins function            |
| **Parameter**           | Input from the user when triggering the build |
| **Environment**         | Variables available to all steps              |
| **when**                | Conditional logic for stages                  |
| **timeout / retry**     | Safety controls to prevent infinite failures  |
| **post**                | Final actions after everything                |
| **junit**               | Publish test results                          |
| **skipDefaultCheckout** | Avoid redundant git clone                     |
| **catchError**          | Force pipeline to finish as SUCCESS           |


Let’s walk through the quiz together clearly and simply.

---

### 🧩 1️⃣ What’s the difference between a **stage** and a **step**?

| Concept   | What it means                                                                                                                                                      | Example                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------- |
| **Stage** | A **section** or **phase** of your pipeline — like a chapter in your automation flow. Used for organizing, visualization, and control (e.g., Build, Test, Deploy). | `stage('Build') { steps { ... } }`                    |
| **Step**  | A **single action** inside a stage — an instruction Jenkins actually executes.                                                                                     | `sh 'echo "Compiling..."'` or `junit 'reports/*.xml'` |

🧠 Analogy:

* **Stage** = a box in your flow diagram.
* **Steps** = the instructions *inside* that box.

---

### 🧩 2️⃣ What does the **`parallel { … }`** block do?

The `parallel` block tells Jenkins:

> “Run these multiple sub-stages **at the same time** instead of one after another.”

Example from Level 2:

```groovy
stage('Test (parallel)') {
  parallel {
    stage('Unit Tests') { ... }
    stage('Integration Tests') { ... }
  }
}
```

🧠 Why it’s powerful:

* Saves time — if both tests take 5 minutes, parallelization still finishes in 5 min, not 10.
* Jenkins displays each branch separately in the UI.

---

### 🧩 3️⃣ How would you change the pipeline to **skip Build for `dev`** but **require it for `staging`/`prod`**?

Use a **`when` condition** that checks the environment variable:

```groovy
stage('Build') {
  when {
    expression { return params.DO_BUILD && env.APP_ENV != 'dev' }
  }
  steps {
    sh '''
      echo "Building for ${APP_ENV}..."
      echo "Build OK"
    '''
  }
}
```

🧠 Explanation:

* The stage only runs if `DO_BUILD == true` **and** the environment is **not** dev.
* For `staging` or `prod`, `APP_ENV != 'dev'`, so it builds.
* For `dev`, it skips automatically.

---

### 🧩 4️⃣ Where do your **artifacts** live after the run (and how do you download them)?

In **GitHub Actions**, after each run:

* The Jenkinsfile creates files (like `dist/app-staging.tar.gz`, `reports/junit/*.xml`) inside the workspace.
* Your GitHub workflow step:

  ```yaml
  - name: Upload outputs
    uses: actions/upload-artifact@v4
    with:
      name: jenkinsfile-outputs
      path: |
        build/**
        reports/junit/*.xml
        dist/**
  ```

  uploads those files as **artifacts** attached to the run.

🧾 To download:

1. Go to **Actions → the run you executed.**
2. On the right side or bottom, find **Artifacts**.
3. Click the name (e.g., `jenkinsfile-outputs`) → it downloads as a `.zip`.
4. Inside, you’ll see `build/`, `reports/`, and `dist/` folders.

✅ Those files are your **hand-off deliverables** — what a real Jenkins would deploy or share.

---

### 🌟 Summary

| Concept               | Key takeaway                                                                             |
| --------------------- | ---------------------------------------------------------------------------------------- |
| **Stage vs Step**     | Stages organize your flow; steps are the commands inside them.                           |
| **Parallel block**    | Runs multiple mini-stages at once to save time.                                          |
| **Conditional Build** | Use `when { expression { ... } }` with environment variables to control stage execution. |
| **Artifacts**         | Stored and downloadable from GitHub Actions → “Artifacts” section on the run page.       |

---

## 🧩 What are **artifacts** in CI/CD?

> **Artifacts = the files your pipeline produces and wants to keep after it finishes.**

They are **outputs** — the *results* of your build, test, or packaging stages.

---

### 🧱 Think of it like this:

When you build something (an app, a report, or even a zip file),
the CI system (like Jenkins or GitHub Actions) runs on a temporary machine.

After the job ends, that machine is deleted 🚮 — unless you tell the system:

> “Hey! Save these files somewhere safe — I’ll need them later.”

Those saved files are called **artifacts**.

---

## 📦 Example in your Level 2 pipeline

Your Jenkinsfile creates these folders and files:

```
build/
  ├─ app.bin
  ├─ config.env
reports/junit/
  ├─ unit.xml
  ├─ integration.xml
dist/
  ├─ app-dev.tar.gz
```

These are the **artifacts** of your run:

* The compiled binary → `app.bin`
* The environment config → `config.env`
* The test reports → `.xml` files
* The packaged release → `app-dev.tar.gz`

They’re valuable because:

* You might want to **download them** and test locally.
* A next pipeline (e.g., “Deploy”) might need them.
* They’re **proof** your build really produced something.

---

## 🧭 Where artifacts live (in GitHub Actions)

In your workflow YAML, you have this step:

```yaml
- name: Upload outputs
  uses: actions/upload-artifact@v4
  with:
    name: jenkinsfile-outputs
    path: |
      build/**
      reports/junit/*.xml
      dist/**
```

That command tells GitHub Actions:

> “Take all files inside `build/`, `reports/junit/`, and `dist/`, and upload them as a downloadable package named `jenkinsfile-outputs`.”

---

## 💻 How to view them

1. Go to your **GitHub repository → Actions → last run**.

2. Scroll to the **bottom right corner** of the run summary.
   You’ll see something like:

   📦 **Artifacts**
   → `jenkinsfile-outputs` (download)

3. Click it — it downloads as a ZIP file.

4. Unzip it locally, and you’ll see all those files your pipeline created.

That ZIP = **your artifacts**.

---

## 🧠 Why artifacts matter in DevOps

| Situation           | Artifact Example          | Purpose                                |
| ------------------- | ------------------------- | -------------------------------------- |
| Building an app     | `.exe`, `.jar`, `.tar.gz` | To deploy or share the build           |
| Running tests       | `.xml` JUnit reports      | To show test results in Jenkins/GitHub |
| Creating docs       | `.pdf`, `.html`           | To publish docs                        |
| Infrastructure runs | `.tfplan`, `.json`        | To preview changes before applying     |
| Data jobs           | `.csv`, `.json`           | To store processed data results        |

Artifacts = the **bridge between stages** in your CI/CD flow:

* Build → Test → Deploy → Release
  Each stage can take the artifact from the previous one.

---

## 🧩 In one sentence

> 🗂️ **Artifacts are the “take-home results” of your automation — files saved from your temporary build environment so you or the next stage can use them later.**

Perfect 🌟 — let’s visualize it step by step, like a DevOps roadmap.

---

# 🧱 Artifacts Flow in a CI/CD Pipeline

*(Simple DevOps View)*

Imagine your pipeline as a **factory line**:
each stage **creates something**, and that “something” is called an **artifact**.

---

## 🚀 Diagram (Text-based)

```
          ┌──────────────────────────────────────┐
          │          Source Code Repo            │
          │   (Your GitHub project, Jenkinsfile) │
          └──────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Build Stage     │
                    │  - Compile code  │
                    │  - Create files  │
                    └─────────────────┘
                              │
                              ▼
                🗂️  Artifact 1: build/app.bin
                🗂️  Artifact 2: build/config.env
                              │
                              ▼
                    ┌─────────────────┐
                    │  Test Stage      │
                    │  - Run tests     │
                    │  - Generate XML  │
                    └─────────────────┘
                              │
                              ▼
                🗂️  Artifact 3: reports/junit/*.xml
                              │
                              ▼
                    ┌─────────────────┐
                    │  Package Stage   │
                    │  - Bundle files  │
                    │  - Create tar.gz │
                    └─────────────────┘
                              │
                              ▼
                🗂️  Artifact 4: dist/app-dev.tar.gz
                              │
                              ▼
                    ┌─────────────────┐
                    │ Upload Artifacts │
                    │  (GitHub Action) │
                    └─────────────────┘
                              │
                              ▼
          💾 Stored in GitHub Actions → Artifacts Section
                              │
                              ▼
            ⬇️ Downloadable ZIP with build/test/package results
```

---

## 💬 Explanation of the flow

| Stage                | What it does                          | Artifact produced                   |
| -------------------- | ------------------------------------- | ----------------------------------- |
| **Build**            | Compiles or creates app files         | `build/app.bin`, `build/config.env` |
| **Test**             | Runs tests, exports results           | `reports/junit/*.xml`               |
| **Package**          | Compresses everything for deployment  | `dist/app-dev.tar.gz`               |
| **Upload Artifacts** | Sends those files to GitHub’s storage | `jenkinsfile-outputs.zip`           |

---

## 🧩 Where you find them

After the pipeline run finishes:

1. Go to **Actions → last run**.
2. Scroll down or look right → you’ll see 📦 **Artifacts**.
3. Click → download → unzip.
   You’ll see:

   ```
   build/app.bin
   build/config.env
   reports/junit/unit.xml
   reports/junit/integration.xml
   dist/app-dev.tar.gz
   ```

These are the **“outputs”** that represent your pipeline’s success.

---

## 🧠 DevOps mindset takeaway

Think of **artifacts as the baton in a relay race**:

* One stage runs → hands off its result (artifact) → next stage uses it.

If you ever build multi-step pipelines like:

```
Build → Test → Deploy → Notify
```

The “Deploy” step will take the *artifact* created in “Build.”






