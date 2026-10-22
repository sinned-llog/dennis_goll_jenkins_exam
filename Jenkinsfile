pipeline {

    agent any

    environment {
        DOCKER_ID = "sinnedllog"
        DOCKER_IMAGE_CAST = "exam_cast"
        DOCKER_IMAGE_MOVIE = "exam_movie"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    pwd
                    ls -la
                '''
            }
        }

        stage('Validate Helm') {
            steps {
                sh '''
                    set -eu

                    helm lint ./helm

                    helm template exam ./helm \
                      --set cast.tag="$DOCKER_TAG" \
                      --set movie.tag="$DOCKER_TAG" \
                      > rendered.yaml

                    kubectl apply \
                      --dry-run=client \
                      -f rendered.yaml

                    echo "Images in rendered Helm manifests:"
                    grep -n "image:" rendered.yaml
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      -t "$DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG" \
                      ./cast-service

                    docker build \
                      -t "$DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG" \
                      ./movie-service
                '''
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }

            steps {
                sh '''
                    set -eu

                    echo "$DOCKER_PASS" | docker login \
                      --username "$DOCKER_ID" \
                      --password-stdin

                    docker push \
                      "$DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG"

                    docker push \
                      "$DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG"
                '''
            }
        }

        stage('Prepare DEV') {
            environment {
                KUBECONFIG = credentials('config')
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }

            steps {
                sh '''
                    set -eu

                    kubectl create namespace dev \
                      --dry-run=client \
                      -o yaml \
                      | kubectl apply -f -

                    kubectl create secret docker-registry regcred \
                      --docker-server=https://index.docker.io/v1/ \
                      --docker-username="$DOCKER_ID" \
                      --docker-password="$DOCKER_PASS" \
                      --namespace dev \
                      --dry-run=client \
                      -o yaml \
                      | kubectl apply -f -
                '''
            }
        }

        stage('Deploy DEV') {
            environment {
                KUBECONFIG = credentials('config')
            }

            steps {
                sh '''
                    set -eu

                    helm upgrade --install exam ./helm \
                      --namespace dev \
                      --set cast.tag="$DOCKER_TAG" \
                      --set movie.tag="$DOCKER_TAG" \
                      --wait \
                      --timeout 2m
                '''
            }
        }

        stage('Check DEV') {
            environment {
                KUBECONFIG = credentials('config')
            }

            steps {
                sh '''
                    set -eu

                    kubectl get pods -n dev
                    kubectl get services -n dev
                    kubectl get deployments -n dev
                    kubectl get statefulsets -n dev

                    kubectl rollout status \
                      statefulset/cast-db \
                      -n dev \
                      --timeout=180s

                    kubectl rollout status \
                      statefulset/movie-db \
                      -n dev \
                      --timeout=180s

                    kubectl rollout status \
                      deployment/cast-service \
                      -n dev \
                      --timeout=180s

                    kubectl rollout status \
                      deployment/movie-service \
                      -n dev \
                      --timeout=180s
                '''
            }
        }

        stage('Test DEV APIs') {
            environment {
                KUBECONFIG = credentials('config')
            }

            steps {
                sh '''
                    set -eu

                    kubectl port-forward \
                      service/cast-service \
                      18002:8000 \
                      -n dev \
                      > /tmp/cast-forward.log 2>&1 &

                    CAST_PID=$!

                    kubectl port-forward \
                      service/movie-service \
                      18001:8000 \
                      -n dev \
                      > /tmp/movie-forward.log 2>&1 &

                    MOVIE_PID=$!

                    cleanup() {
                        kill "$CAST_PID" "$MOVIE_PID" 2>/dev/null || true
                    }

                    trap cleanup EXIT

                    sleep 5

                    curl --fail --show-error \
                      http://127.0.0.1:18002/api/v1/casts

                    curl --fail --show-error \
                      http://127.0.0.1:18001/api/v1/movies
                '''
            }
        }
    }

    post {
        success {
            echo "DEV deployment and API tests succeeded."
        }

        failure {
            echo "Pipeline failed. Check the failed stage."
        }
    }
}