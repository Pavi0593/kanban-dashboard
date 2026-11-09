pipeline { 
  agent any 
  options { timestamps(); disableConcurrentBuilds(); timeout(time: 20, unit: 'MINUTES') } 
  environment { 
    APP_NAME = 'kanban-dashboard' 
    DOCKERHUB_USER = ' pavi0593' 
    REGISTRY_CREDENTIALS_ID = 'dockerhub-creds' 
    APP_PORT = '3000' 
    CONTAINER_PORT = '8080' 
  } 
  stages { 
    stage('Checkout') { 
      steps { 
        checkout scm 
        script { 
          env.GIT_SHORT_SHA = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim() 
          env.IMAGE_TAG = "${BUILD_NUMBER}-${GIT_SHORT_SHA}" 
          env.IMAGE = "${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}" 
        } 
      } 
    } 
    stage('Build') { steps { sh 'docker build --pull -t ${IMAGE} .' } } 
    stage('Push') {
    steps {
        script {
            echo "Image being pushed: ${env.IMAGE}"
        }

        withCredentials([
            usernamePassword(
                credentialsId: env.REGISTRY_CREDENTIALS_ID,
                usernameVariable: 'REG_USER',
                passwordVariable: 'REG_TOKEN'
            )
        ]) {
            sh '''
                set +x
                echo "$REG_TOKEN" | docker login -u "$REG_USER" --password-stdin
                echo "Pushing image: $IMAGE"
                docker push "$IMAGE"
                docker logout
            '''
        }
    }
}
stage('Deploy') { 
      steps { 
        sh ''' 
          set -eu 
          docker pull "$IMAGE" 
          docker inspect -f '{{.Config.Image}}' "$APP_NAME" > .previous-image 2>/dev/null || true 
          docker rm -f "$APP_NAME" 2>/dev/null || true 
          docker run -d --name "$APP_NAME" --restart unless-stopped --memory=384m --cpus=0.75 -p "${APP_PORT}:${CONTAINER_PORT}" "$IMAGE" 
        ''' 
      } 
    } 
    stage('Health Check') { 
      steps { 
        sh ''' 
          set -eu 
          for i in $(seq 1 20); do 
            STATUS=$(docker inspect -f '{{.State.Health.Status}}' "$APP_NAME" 2>/dev/null || echo missing) 
            [ "$STATUS" = healthy ] && break 
            [ "$STATUS" = unhealthy ] && exit 1 
            sleep 3 
          done 
          [ "$(docker inspect -f '{{.State.Health.Status}}' "$APP_NAME")" = healthy ] 
          curl -fsS "http://127.0.0.1:${APP_PORT}/health" 
        ''' 
      } 
    } 
  } 
  post { 
    success { sh 'rm -f .previous-image' } 
    failure { 
      sh ''' 
        set +e 
        if [ -s .previous-image ]; then 
          PREVIOUS_IMAGE=$(cat .previous-image) 
          docker rm -f "$APP_NAME" 2>/dev/null || true 
          docker run -d --name "$APP_NAME" --restart unless-stopped --memory=384m --cpus=0.75 -p "${APP_PORT}:${CONTAINER_PORT}" "$PREVIOUS_IMAGE" 
        fi 
      ''' 
    } 
  } 
} 