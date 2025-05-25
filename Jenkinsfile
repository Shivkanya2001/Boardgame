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
        withCredentials([string(credentialsId: 'k8config-secret', variable: 'KUBECONFIG_CONTENT')]) {
          writeFile file: 'kubeconfig.yaml', text: KUBECONFIG_CONTENT
          sh 'kubectl apply -f deployment-service.yaml --kubeconfig=kubeconfig.yaml'
        }
      }
    }

    stage('Verify the Deployment') {
      steps {
        withCredentials([string(credentialsId: 'k8config-secret', variable: 'KUBECONFIG_CONTENT')]) {
          writeFile file: 'kubeconfig.yaml', text: KUBECONFIG_CONTENT
          sh '''
            kubectl get pods -n webapps --kubeconfig=kubeconfig.yaml
            kubectl get svc -n webapps --kubeconfig=kubeconfig.yaml
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
