pipeline {
  agent any

  environment {
    DOCKER_IMAGE = 'docker.io/bayesaermbow/java17-render-app'
  }

  options {
    timestamps()
    // ansiColor('xterm') // décommente si le plugin AnsiColor est installé
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set Version') {
      steps {
        script {
          // Fabrique une version: date + short SHA
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
        sh "docker build -t ${DOCKER_IMAGE}:${env.VERSION} -t ${DOCKER_IMAGE}:latest ."
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          // login : garder des quotes simples pour conserver $DOCKER_*
          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
          '''
          // push avec interpolation Groovy pour DOCKER_IMAGE / VERSION
          sh """
            set -e
            docker push ${DOCKER_IMAGE}:${env.VERSION}
            docker push ${DOCKER_IMAGE}:latest
            docker logout
          """
        }
      }
    }

    stage('Deploy on Render') {
      steps {
        withCredentials([
          string(credentialsId: 'render-hook',    variable: 'RENDER_DEPLOY_HOOK'),
          string(credentialsId: 'render-app-url', variable: 'APP_URL')
        ]) {
          // 1) Déclenche le déploiement Render (avec petit retry)
          sh 'curl -fsSL -X POST "$RENDER_DEPLOY_HOOK" || (sleep 5 && curl -fsSL -X POST "$RENDER_DEPLOY_HOOK")'

          // 2) Attend que l’app réponde (max ~5 min)
          sh '''
            set -e
            echo "Attente que l'app démarre sur $APP_URL ..."
            for i in $(seq 1 60); do
              # /actuator/health si dispo, sinon la racine /
              if curl -fsS --max-time 5 "$APP_URL/actuator/health" >/dev/null 2>&1 || \
                 curl -fsS --max-time 5 "$APP_URL"               >/dev/null 2>&1; then
                echo "✅ App UP : $APP_URL"
                exit 0
              fi
              sleep 5
            done
            echo "❌ App indisponible après attente : $APP_URL"
            exit 1
          '''
        }
      }
    }

  }

  post {
    success { echo " OK — Image: ${DOCKER_IMAGE}:${env.VERSION} — Déploiement Render déclenché + app UP." }
    failure { echo "Échec du pipeline." }
  }
}
