pipeline {
  agent any

  tools {
    jdk 'jdk17'
    maven 'maven3'
  }

  environment {
    SCANNER_HOME = tool 'sonar-scanner'
  }

  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Shivkanya2001/Boardgame'
      }
    }

    stage('Compile') {
      steps {
        sh 'mvn compile'
      }
    }

    stage('Test') {
      steps {
        sh 'mvn test'
      }
    }

    stage('File System Scan') {
      steps {
        sh 'trivy fs --format table -o $WORKSPACE/trivy-fs-report.html . || true'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar') {
          sh '''
            $SCANNER_HOME/bin/sonar-scanner \
              -Dsonar.projectName=BoardGame \
              -Dsonar.projectKey=BoardGame \
              -Dsonar.java.binaries=target
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 2, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
          }
        }
      }
    }

    stage('Build') {
      steps {
        sh 'mvn package -DskipTests'
      }
    }

    stage('Publish To Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
          configFileProvider([configFile(fileId: 'global-settings', variable: 'MAVEN_SETTINGS')]) {
            withMaven(
              mavenSettingsConfig: 'global-settings',
              globalMavenSettingsConfig: 'global-settings',
              jdk: 'jdk17',
              maven: 'maven3'
            ) {
              sh 'mvn deploy -DskipTests'
            }
          }
        }
      }
    }

    stage('Build & Tag Docker Image') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
            sh 'docker build -t shivkanyadoiphode/boardgame:latest .'
          }
        }
      }
    }

    stage('Docker Image Scan') {
      steps {
        sh 'trivy image --format table -o $WORKSPACE/trivy-image-report.html shivkanyadoiphode/boardgame:latest || true'
      }
    }

    stage('Push Docker Image') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
            sh 'docker push shivkanyadoiphode/boardgame:latest'
          }
        }
      }
    }

    stage('Deploy To Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'k8config-file', variable: 'KUBECONFIG')]) {
          sh '''
            echo "[INFO] Ensuring 'webapps' namespace exists..."
            kubectl create namespace webapps --kubeconfig=$KUBECONFIG || true

            echo "[INFO] Current working directory:"
            pwd
            echo "[INFO] Listing files to confirm deployment-service.yaml exists:"
            ls -l

            echo "[INFO] Applying Kubernetes deployment (with timeout)..."
            kubectl apply --timeout=30s -f deployment-service.yaml --kubeconfig=$KUBECONFIG --validate=false || echo "[WARN] kubectl apply may have timed out"

            echo "[INFO] Validating applied resources..."
            kubectl get deployments -n webapps --kubeconfig=$KUBECONFIG || true
            kubectl get services -n webapps --kubeconfig=$KUBECONFIG || true
          '''
        }
      }
    }

    stage('Verify the Deployment') {
      steps {
        withCredentials([file(credentialsId: 'k8config-file', variable: 'KUBECONFIG')]) {
          sh '''
            echo "[INFO] Verifying deployed resources..."
            kubectl get pods -n webapps --kubeconfig=$KUBECONFIG || true
            kubectl get svc -n webapps --kubeconfig=$KUBECONFIG || true

            echo "[INFO] Waiting for rollout to complete..."
            kubectl rollout status deployment/boardgame-deployment -n webapps --timeout=120s --kubeconfig=$KUBECONFIG || {
              echo "[ERROR] Rollout did not complete in time."
              exit 1
            }

            echo "[INFO] Describing pods for debug..."
            kubectl describe pods -n webapps --kubeconfig=$KUBECONFIG || true
          '''
        }
      }
    }
  }

  post {
    always {
      script {
        def status = currentBuild.result ?: 'SUCCESS'
        def body = """
        <html><body>
        <h2>${env.JOB_NAME} - Build #${env.BUILD_NUMBER}</h2>
        <h3 style='color:white;background-color:${status == 'SUCCESS' ? 'green' : 'red'};padding:10px;'>Pipeline Status: ${status}</h3>
        <p><a href='${env.BUILD_URL}'>View Console Output</a></p>
        </body></html>
        """
        emailext (
          subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${status}",
          body: body,
          to: 'shivani.doiphode2001@gmail.com',
          from: 'jenkins@example.com',
          mimeType: 'text/html',
          attachmentsPattern: '**/trivy-image-report.html'
        )
      }
    }
  }
}
