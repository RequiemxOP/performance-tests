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
          branches: [[name: '*/main']],   // UPDATED BRANCH
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
          def found = sh(
            script: "ls -1t Vaults/*/*.jmx 2>/dev/null | head -n 1",
            returnStdout: true
          ).trim()

          if (!found) {
            error "No JMX found in Vaults/*/"
          }

          LATEST_JMX = found
          VAULT = found.split('/')[1]

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
          def vault = readFile("vault.txt").trim()

          RUN_DIR = "${BASE_RESULTS}/${vault}/${BUILD_NUMBER}-${ts}"
          sh "mkdir -p '${RUN_DIR}'"

          // Copy files into run directory
          sh "cp -r data '${RUN_DIR}/data'"
          sh "cp '${LATEST_JMX}' '${RUN_DIR}/testplan.jmx'"

          writeFile file: "run_dir.txt", text: RUN_DIR
        }
      }
    }

    stage('Normalize JMX Paths') {
      steps {
        sh '''
        set -eu
        RUN_DIR=$(cat run_dir.txt)
        JMX="${RUN_DIR}/testplan.jmx"

        # Normalize Windows paths
        sed -i 's#\\\\#/#g' "$JMX"
        sed -i 's#[A-Za-z]:/##g' "$JMX"

        # Fix CSV references
        for f in ${RUN_DIR}/data/*; do
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
        RUN_DIR=$(cat run_dir.txt)

        ${JMETER_HOME}/jmeter -n \
          -t "${RUN_DIR}/testplan.jmx" \
          -l "${RUN_DIR}/results.jtl" \
          -j "${RUN_DIR}/jmeter.log" \
          -e -o "${RUN_DIR}/html"
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
