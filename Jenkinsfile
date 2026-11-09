pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        APP_NAME = 'kanban-dashboard'

        // IMPORTANT: no space before pavi0593
        DOCKERHUB_USER = 'pavi0593'

        REGISTRY_CREDENTIALS_ID = 'dockerhub-creds'

        // Browser accesses application on EC2 port 3000
        APP_PORT = '3000'

        // NGINX inside container listens on 8080
        CONTAINER_PORT = '8080'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.GIT_SHORT_SHA = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG =
                        "${env.BUILD_NUMBER}-${env.GIT_SHORT_SHA}"

                    env.IMAGE =
                        "${env.DOCKERHUB_USER}/${env.APP_NAME}:${env.IMAGE_TAG}".trim()

                    echo "Git SHA: ${env.GIT_SHORT_SHA}"
                    echo "Image Tag: ${env.IMAGE_TAG}"
                    echo "Docker Image: ${env.IMAGE}"
                }
            }
        }


        stage('Build') {
            steps {
                sh '''
                    set -eu

                    echo "Building Docker image:"
                    echo "$IMAGE"

                    test -n "$IMAGE"

                    docker build \
                      --pull \
                      -t "$IMAGE" \
                      .

                    echo "Checking image exists..."

                    docker image inspect "$IMAGE" > /dev/null

                    echo "Docker image build completed."
                '''
            }
        }


        stage('Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.REGISTRY_CREDENTIALS_ID,
                        usernameVariable: 'REG_USER',
                        passwordVariable: 'REG_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Pushing image:"
                        echo "$IMAGE"

                        test -n "$IMAGE"

                        set +x

                        echo "$REG_TOKEN" | \
                          docker login \
                          -u "$REG_USER" \
                          --password-stdin

                        docker push "$IMAGE"

                        docker logout

                        echo "Docker image pushed successfully."
                    '''
                }
            }
        }


        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    echo "Deploying image:"
                    echo "$IMAGE"

                    docker pull "$IMAGE"

                    # Remove previous-image record from an older build
                    rm -f .previous-image

                    # If an old container exists,
                    # remember its image before replacing it
                    if docker inspect "$APP_NAME" \
                       > /dev/null 2>&1; then

                        echo "Existing deployment found."

                        docker inspect \
                          -f '{{.Config.Image}}' \
                          "$APP_NAME" \
                          > .previous-image

                        echo "Previous image:"
                        cat .previous-image
                    else
                        echo "First deployment. No previous container found."
                    fi

                    # Remove current application container
                    docker rm -f "$APP_NAME" \
                      2>/dev/null || true

                    # Start new version
                    docker run -d \
                      --name "$APP_NAME" \
                      --restart unless-stopped \
                      --memory=384m \
                      --cpus=0.75 \
                      -p "${APP_PORT}:${CONTAINER_PORT}" \
                      "$IMAGE"

                    echo "New container started."
                '''
            }
        }


        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    echo "Waiting for Docker HEALTHCHECK..."

                    for i in $(seq 1 20); do

                        STATUS=$(docker inspect \
                          -f '{{.State.Health.Status}}' \
                          "$APP_NAME" \
                          2>/dev/null || echo missing)

                        echo "Health attempt $i: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            echo "Container is healthy."
                            break
                        fi

                        if [ "$STATUS" = "unhealthy" ]; then
                            echo "Container is unhealthy."

                            echo "Container logs:"
                            docker logs "$APP_NAME" || true

                            exit 1
                        fi

                        sleep 3
                    done

                    FINAL_STATUS=$(docker inspect \
                      -f '{{.State.Health.Status}}' \
                      "$APP_NAME")

                    echo "Final Docker health:"
                    echo "$FINAL_STATUS"

                    test "$FINAL_STATUS" = "healthy"

                    echo "Testing application health endpoint..."

                    curl -fsS \
                      "http://127.0.0.1:${APP_PORT}/health"

                    echo ""
                    echo "Application HTTP health check passed."
                '''
            }
        }
    }


    post {

        success {

            echo "CI/CD pipeline completed successfully."

            sh '''
                rm -f .previous-image

                echo "Running application container:"
                docker ps \
                  --filter "name=$APP_NAME"

                echo "Deployed image:"
                docker inspect \
                  -f '{{.Config.Image}}' \
                  "$APP_NAME"

                echo "Docker health:"
                docker inspect \
                  -f '{{.State.Health.Status}}' \
                  "$APP_NAME"

                echo "Resource limits:"
                docker inspect \
                  --format='Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}' \
                  "$APP_NAME"
            '''
        }


        failure {

            echo "Pipeline failed. Checking rollback availability."

            sh '''
                set +e

                if [ -s .previous-image ]; then

                    PREVIOUS_IMAGE=$(cat .previous-image)

                    echo "Previous image found:"
                    echo "$PREVIOUS_IMAGE"

                    echo "Removing failed deployment..."

                    docker rm -f "$APP_NAME" \
                      2>/dev/null || true

                    echo "Pulling previous working image..."

                    docker pull "$PREVIOUS_IMAGE" || true

                    echo "Starting previous working version..."

                    docker run -d \
                      --name "$APP_NAME" \
                      --restart unless-stopped \
                      --memory=384m \
                      --cpus=0.75 \
                      -p "${APP_PORT}:${CONTAINER_PORT}" \
                      "$PREVIOUS_IMAGE"

                    echo "Rollback container started."

                    sleep 5

                    echo "Rollback container status:"

                    docker ps \
                      --filter "name=$APP_NAME"

                else

                    echo "No previous deployment available for rollback."

                fi
            '''
        }


        always {

            sh '''
                docker logout \
                  >/dev/null 2>&1 || true
            '''
        }
    }
}