
# ✅ Level 2 Jenkinsfile (drop-in)

Copy–paste this over your current `Jenkinsfile`.

```groovy
// Level 2: Parallel tests + artifact hand-off + environment selection
// Safe for Jenkinsfile Runner (no timestamps()/archiveArtifacts()).

pipeline {
  agent any

  options {
    // Avoid Jenkins' default SCM checkout (GitHub Actions already does it)
    skipDefaultCheckout(true)

    // Keep training runs green even if Runner marks UNSTABLE
    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS')
  }

  // ── Parameters ──────────────────────────────────────────────────────────────
  parameters {
    string(name: 'NAME',     defaultValue: 'World', description: 'Who to greet')
    booleanParam(name: 'DO_BUILD', defaultValue: true, description: 'Run Build stage?')
    string(name: 'TARGET_ENV', defaultValue: 'dev',  description: 'Environment: dev|staging|prod')
  }

  // ── Global environment ─────────────────────────────────────────────────────
  environment {
    APP_ENV = "${params.TARGET_ENV}"    // e.g., dev/staging/prod
    BUILD_DIR = "build"
    REPORTS_DIR = "reports/junit"
    DIST_DIR = "dist"
  }

  stages {

    // 1) Init: prepare workspace and config per environment
    stage('Init') {
      steps {
        echo "Hi ${params.NAME}! APP_ENV=${env.APP_ENV}"
        sh '''
          set -eu
          mkdir -p "${BUILD_DIR}" "${REPORTS_DIR}" "${DIST_DIR}"
          # Simulate environment-specific config
          cat > "${BUILD_DIR}/config.env" <<EOF
          APP_ENV=${APP_ENV}
          API_URL=https://api.${APP_ENV}.example.local
          FEATURE_FLAG=${APP_ENV}_features_on
          EOF
          echo "Init OK"
        '''
      }
    }

    // 2) Build (optional)
    stage('Build') {
      when { expression { return params.DO_BUILD } }
      steps {
        timeout(time: 1, unit: 'MINUTES') {
          retry(1) {
            sh '''
              set -eu
              echo "Compiling sources for ${APP_ENV}..."
              # Simulate build outputs
              echo "binary-for-${APP_ENV}" > "${BUILD_DIR}/app.bin"
              echo "Build OK"
            '''
          }
        }
      }
    }

    // 3) Test in parallel (fast feedback)
    stage('Test (parallel)') {
      parallel {
        stage('Unit Tests') {
          agent any
          steps {
            sh '''
              set -eu
              # Simulate unit tests -> JUnit XML
              cat > "${REPORTS_DIR}/unit.xml" <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite name="unit" tests="3" failures="0" errors="0" skipped="0" time="0.02">
                <testcase classname="u.Add" name="adds" time="0.001"/>
                <testcase classname="u.Sub" name="subs" time="0.001"/>
                <testcase classname="u.Mix" name="mixes" time="0.001"/>
              </testsuite>
              XML
            '''
          }
        }

        stage('Integration Tests') {
          agent any
          steps {
            sh '''
              set -eu
              # Simulate integration tests -> JUnit XML
              cat > "${REPORTS_DIR}/integration.xml" <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite name="integration" tests="2" failures="0" errors="0" skipped="0" time="0.03">
                <testcase classname="it.API" name="responds" time="0.002"/>
                <testcase classname="it.DB"  name="reads"    time="0.002"/>
              </testsuite>
              XML
            '''
          }
        }
      }
    }

    // 4) Collect & publish test results
    stage('Test Report') {
      steps {
        // Let Jenkins parse both unit + integration XMLs
        junit testResults: "${REPORTS_DIR}/*.xml", allowEmptyResults: true
        sh 'echo "Merged JUnit OK"'
      }
    }

    // 5) Package (artifact hand-off)
    stage('Package') {
      steps {
        sh '''
          set -eu
          # Bundle everything needed for deployment/distribution
          tar -czf "${DIST_DIR}/app-${APP_ENV}.tar.gz" \
            "${BUILD_DIR}/app.bin" \
            "${BUILD_DIR}/config.env" \
            "${REPORTS_DIR}"
          echo "Package created at ${DIST_DIR}/app-${APP_ENV}.tar.gz"
        '''
      }
    }
  }

  post {
    always {
      echo "Level 2 finished for APP_ENV=${APP_ENV}"
    }
  }
}
```

---

# 🧠 What you just added (and why)

## 1) **Environment selection**

* `TARGET_ENV` parameter + `APP_ENV` variable let you simulate **dev/staging/prod**.
* Your build and package steps read `APP_ENV` to change behavior/filenames.

## 2) **Parallel testing**

* `stage('Test (parallel)') { parallel { ... } }` runs **Unit** and **Integration** at the same time.
* This is a real DevOps skill: make feedback faster by splitting test types.

## 3) **Artifact hand-off**

* We package outputs into `dist/app-<env>.tar.gz`.
* In real Jenkins, you’d often use `archiveArtifacts` or publish to storage;
  here we keep it simple and compatible with Runner.
* Your GitHub Action already **uploads artifacts** from the workspace, so you’ll be able to download `dist/app-*.tar.gz`.

## 4) **Stable training runs**

* `skipDefaultCheckout(true)` avoids the Runner’s special checkout (which caused UNSTABLE before).
* `catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS')` ensures the overall build is green even if Runner marks it UNSTABLE for non-fatal reasons.

---

# ▶ How to run it

You already have a workflow that runs your Jenkinsfile (the “Run Jenkinsfile (Jenkinsfile Runner)” one).
Run it from **Actions → Run workflow** and (optionally) set:

* `NAME = Eliyahu`
* `TARGET_ENV = staging`
* `DO_BUILD = true`

When it finishes:

* Open the job logs → you’ll see **Unit Tests** and **Integration Tests** ran **in parallel**.
* On the run page, download the artifact; inside you’ll have:

  * `dist/app-<env>.tar.gz`
  * JUnit XMLs in `reports/junit/`

---

# 🧩 Mini-quiz (cement the concept)

1. What’s the difference between a **stage** and a **step**?
2. What does the **`parallel { … }`** block do?
3. How would you change the pipeline to **skip** Build for `dev` but **require** it for `staging`/`prod`?
4. Where do your **artifacts** live after the run (and how do you download them)?


