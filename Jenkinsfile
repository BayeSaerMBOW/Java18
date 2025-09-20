pipeline {
  agent any

  environment {
    DOCKER_IMAGE = 'docker.io/bayesaermbow/java17-render-app'
  }

  options {
    timestamps()
    // ansiColor('xterm') // ← à réactiver uniquement si le plugin AnsiColor est installé
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set Version') {
      steps {
        script {
          // Récupère date et SHA via deux commandes séparées (évite tout $() dans une chaîne Groovy)
          def ts  = sh(script: 'date +%Y%m%d-%H%M%S',        returnStdout: true).trim()
          def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.VERSION = "${ts}-${sha}"
          echo "Version: ${env.VERSION}"
        }
      }
    }

    stage('Build & Test (Maven)') {
      steps {
        sh 'mvn -B -Dmaven.test.failure.ignore=false clean package'
      }
    }

    stage('Docker Build') {
      steps {
        sh """
          docker build -t ${DOCKER_IMAGE}:${env.VERSION} -t ${DOCKER_IMAGE}:latest .
        """
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          // 1) login : garder un bloc en guillemets simples pour laisser $DOCKER_* au shell
          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
          '''
          // 2) push : ici on laisse Groovy interpoler DOCKER_IMAGE / env.VERSION
          sh """
            set -e
            docker push ${DOCKER_IMAGE}:${env.VERSION}
            docker push ${DOCKER_IMAGE}:latest
            docker logout
          """
        }
      }
    }


  }
//sfdsdddsdsds
  post {
    success { echo "✅ OK — Image: ${DOCKER_IMAGE}:${env.VERSION} — Déploiement Render déclenché." }
    failure { echo "❌ Échec du pipeline." }
  }
}
