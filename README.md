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



