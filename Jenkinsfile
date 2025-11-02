// Level 1: Minimal, plugin-light Declarative pipeline
// Works with Jenkinsfile Runner in GitHub Actions
// Key ideas: parameters, environment, stages, when/timeout/retry, post

pipeline {
  agent any

  // Avoid runner's special checkout that caused UNSTABLE
  options { skipDefaultCheckout(true) }

  // 1) Parameters: pass custom values at runtime
  parameters {
    string(name: 'NAME', defaultValue: 'World', description: 'Who to greet')
    booleanParam(name: 'DO_BUILD', defaultValue: true, description: 'Run Build stage?')
  }

  // 2) Environment variables available to all stages
  environment {
    APP_ENV = 'dev'
  }

  stages {
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

    stage('Build') {
      when { expression { return params.DO_BUILD } } // run only if DO_BUILD=true
      steps {
        // Show timeout/retry without failing the demo run
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

    stage('Test') {
      steps {
        // Create a tiny demo JUnit report (keeps run green)
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
        // Jenkinsfile Runner prepackaged includes JUnit step
        junit testResults: 'reports/junit/*.xml', allowEmptyResults: true
      }
    }
  }

  post {
    success {
      echo 'Pipeline SUCCESS ✅'
    }
    always {
      echo 'Level 1 finished.'
    }
  }
}
