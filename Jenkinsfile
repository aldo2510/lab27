pipeline {
  agent any

  environment {
    APP_NAME = 'hello-lab27'
    IMAGE = "hello-lab27:${env.BUILD_NUMBER}"
    // URL real del Pushgateway
    PUSHGATEWAY_URL = 'http://34.125.165.197:9091'
  }

  stages {
    stage('Start metrics') {
      steps {
        sh '''
          set -e
          J_SAFE=$(echo "${JOB_NAME}" | tr ' /' '__')
          BR=${BRANCH_NAME:-main}
          ts=$(date +%s)
          echo $ts > .start_ts
          cat <<EOF | curl --retry 3 --silent --show-error --data-binary @- ${PUSHGATEWAY_URL}/metrics/job/jenkins_pipeline/jenkins_job/$J_SAFE/build/${BUILD_NUMBER}/branch/${BR}
# TYPE pipeline_start_timestamp gauge
pipeline_start_timestamp ${ts}
# TYPE pipeline_status gauge
pipeline_status 1
EOF
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // Usa TU Dockerfile para testear (sin bind-mounts)
    stage('Unit tests') {
      steps {
        dir('app') {
          sh """
            docker build -t ${IMAGE}-test .
            docker run --rm ${IMAGE}-test npm test
          """
        }
      }
    }

    stage('Build image') {
      steps {
        dir('app') {
          sh "docker build -t ${IMAGE} ."
        }
      }
    }

    stage('Security scan (Trivy)') {
      steps {
        sh '''
          if ! command -v trivy >/dev/null 2>&1; then
            echo "Using Dockerized Trivy..."
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 0 --severity HIGH,CRITICAL --no-progress ${IMAGE} | tee trivy-report.txt
          else
            trivy image --exit-code 0 --severity HIGH,CRITICAL --no-progress ${IMAGE} | tee trivy-report.txt
          fi
        '''
        archiveArtifacts artifacts: 'trivy-report.txt', onlyIfSuccessful: false
      }
    }
  }

  post {
    success {
      sh '''
        set -e
        end=$(date +%s)
        start=$(cat .start_ts 2>/dev/null || echo $end)
        duration=$((end-start))
        J_SAFE=$(echo "${JOB_NAME}" | tr ' /' '__')
        BR=${BRANCH_NAME:-main}
        cat <<EOF | curl --retry 3 --silent --show-error --data-binary @- ${PUSHGATEWAY_URL}/metrics/job/jenkins_pipeline/jenkins_job/$J_SAFE/build/${BUILD_NUMBER}/branch/${BR}
# TYPE pipeline_end_timestamp gauge
pipeline_end_timestamp ${end}
# TYPE pipeline_status gauge
pipeline_status 0
# TYPE pipeline_duration_seconds gauge
pipeline_duration_seconds ${duration}
EOF
      '''
    }
    failure {
      sh '''
        set -e
        end=$(date +%s)
        start=$(cat .start_ts 2>/dev/null || echo $end)
        duration=$((end-start))
        J_SAFE=$(echo "${JOB_NAME}" | tr ' /' '__')
        BR=${BRANCH_NAME:-main}
        cat <<EOF | curl --retry 3 --silent --show-error --data-binary @- ${PUSHGATEWAY_URL}/metrics/job/jenkins_pipeline/jenkins_job/$J_SAFE/build/${BUILD_NUMBER}/branch/${BR}
# TYPE pipeline_end_timestamp gauge
pipeline_end_timestamp ${end}
# TYPE pipeline_status gauge
pipeline_status 2
# TYPE pipeline_duration_seconds gauge
pipeline_duration_seconds ${duration}
EOF
      '''
    }
    always {
      echo "Build URL: ${env.BUILD_URL}"
    }
  }
}
