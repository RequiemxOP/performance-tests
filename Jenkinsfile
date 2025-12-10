pipeline {
  agent any

  environment {
    JMETER_HOME = "/opt/jmeter/bin"
    BASE_RESULTS = "/opt/jmeter/results"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/PersonalVault']],
          userRemoteConfigs: [[
            url: 'https://github.com/RequiemxOP/performance-tests.git',
            credentialsId: 'github-https-token'
          ]]
        ])
      }
    }

    stage('Detect Latest JMX') {
      steps {
        script {
          // Detect vault folder that has a JMX file
          LATEST_JMX = sh(
            script: "ls -1t Vaults/*/*.jmx | head -n 1",
            returnStdout: true
          ).trim()

          if (!LATEST_JMX) {
            error "No JMX found in Vaults/*/"
          }

          VAULT = LATEST_JMX.split('/')[1]

          echo "Detected vault: ${VAULT}"
          echo "JMX: ${LATEST_JMX}"

          writeFile file: "latest_jmx.txt", text: LATEST_JMX
          writeFile file: "vault.txt", text: VAULT
        }
      }
    }

    stage('Prepare Results Directory') {
      steps {
        script {
          def ts = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()
          VAULT = readFile("vault.txt").trim()

          RUN_DIR = "${BASE_RESULTS}/${VAULT}/${BUILD_NUMBER}-${ts}"
          sh "mkdir -p '${RUN_DIR}'"

          writeFile file: "run_dir.txt", text: RUN_DIR
        }
      }
    }

    stage('Normalize JMX Paths') {
      steps {
        sh '''
        set -eu

        JMX=$(cat latest_jmx.txt)

        # Normalize paths
        sed -i 's#\\\\#/#g' "$JMX"
        sed -i 's#[A-Za-z]:/##g' "$JMX"

        for f in data/*; do
          name=$(basename "$f")
          sed -i -E "s#[A-Za-z0-9_./-]*/${name}#data/${name}#g" "$JMX"
        done

        echo "--- After Normalization ---"
        grep -E "data/.+" -n "$JMX" || true
        '''
      }
    }

    stage('Run JMeter') {
      steps {
        sh '''
        set -eu
        JMX=$(cat latest_jmx.txt)
        RUN_DIR=$(cat run_dir.txt)

        ${JMETER_HOME}/jmeter -n \
          -t "$JMX" \
          -l "$RUN_DIR/results.jtl" \
          -j "$RUN_DIR/jmeter.log" \
          -e -o "$RUN_DIR/html"
        '''
      }
    }

    stage('Publish Report') {
      steps {
        script {
          def RUN_DIR = readFile("run_dir.txt").trim()
          sh """
            rm -rf '${WORKSPACE}/html-report'
            mkdir -p '${WORKSPACE}/html-report'
            cp -r '${RUN_DIR}/html/'* '${WORKSPACE}/html-report/' || true
          """
        }

        publishHTML(target: [
          reportDir: 'html-report',
          reportFiles: 'index.html',
          reportName: "JMeter HTML Report",
          keepAll: true
        ])
      }
    }
  }

  post {
    always { echo "Pipeline finished." }
  }
}
