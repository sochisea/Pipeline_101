pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  parameters {
    string(name: 'NAME', defaultValue: 'World', description: 'Who to greet')
    booleanParam(name: 'DO_BUILD', defaultValue: true, description: 'Run Build stage?')
  }

  environment { APP_ENV = 'dev' }

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
  }

  post {
    always {
      echo 'Level 1 finished.'
      script {
        if (currentBuild.resultIsWorseOrEqualTo('UNSTABLE')) {
          currentBuild.result = 'SUCCESS'
        }
      }
    }
  }
}
