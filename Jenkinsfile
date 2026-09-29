pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }
  environment {
    DOCKERHUB_USER = 'tusharbisen0108'
    IMAGE          = "${DOCKERHUB_USER}/expensesapp"
    IMAGE_TAG      = "${BUILD_NUMBER}"
    APP_PORT       = '8082'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Secret Scan (Gitleaks)') {
      steps {
        sh 'docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:latest dir /repo --redact --verbose'
      }
    }

    stage('Build & Unit Test') {
      steps { sh 'chmod +x mvnw && ./mvnw -B clean verify' }
      post { always { junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml' } }
    }

    stage('SAST (SonarQube)') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=expenses-tracker'
        }
      }
    }
    stage('Quality Gate') {
      steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
    }

    stage('Dependency & IaC Scan (Trivy fs)') {
      steps {
        sh 'trivy fs --scanners vuln,secret,misconfig --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed .'
      }
    }

    stage('Docker Build') {
      steps { sh 'docker build -t $IMAGE:$IMAGE_TAG .' }
    }

    stage('Image Scan (Trivy)') {
      steps {
        sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed $IMAGE:$IMAGE_TAG'
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$IMAGE_TAG
            docker tag  $IMAGE:$IMAGE_TAG $IMAGE:latest
            docker push $IMAGE:latest
          '''
        }
      }
    }

    stage('Deploy (Compose)') {
      steps {
        withCredentials([
          string(credentialsId: 'db-root-password', variable: 'DB_ROOT_PASSWORD'),
          string(credentialsId: 'db-password',      variable: 'DB_PASSWORD')
        ]) {
          sh '''
            docker compose pull mainapp
            docker compose up -d --remove-orphans
            for i in $(seq 1 30); do
              s=$(docker inspect -f '{{.State.Health.Status}}' expensetracker 2>/dev/null || true)
              [ "$s" = "healthy" ] && exit 0
              sleep 5
            done
            docker compose logs --tail=100; exit 1
          '''
        }
      }
    }

    stage('DAST (OWASP ZAP baseline)') {
      steps {
        sh '''
          docker pull ghcr.io/zaproxy/zaproxy:stable
          docker run --rm --user root --network host -v "$PWD:/zap/wrk:rw" \
            ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
            -t http://localhost:$APP_PORT -r zap-report.html || [ $? -eq 2 ]
        '''
        // exit 2 = warnings only (tolerated); exit 1 = FAIL-level findings (fails the build)
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
      sh 'docker logout || true; docker image prune -f'
    }
  }
}
