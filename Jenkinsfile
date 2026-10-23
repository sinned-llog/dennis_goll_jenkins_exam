// helper functions to prepare und deploy each stage
def prepareEnvironment(String environmentName) {
    withCredentials([
        file(
            credentialsId: 'config',
            variable: 'KUBECONFIG'
        ),

        string(
            credentialsId: 'DOCKER_HUB_PASS',
            variable: 'DOCKER_PASS'
        )
    ]) {
        withEnv([
            "TARGET_NAMESPACE=${environmentName}"
        ]) {
            sh'''
                set -eu

                echo "Preparing namespace: $TARGET_NAMESPACE"

                kubectl create namespace "$TARGET_NAMESPACE" \
                      --dry-run=client \
                      -o yaml \
                      | kubectl apply -f -

                    kubectl create secret docker-registry regcred \
                      --docker-server=https://index.docker.io/v1/ \
                      --docker-username="$DOCKER_ID" \
                      --docker-password="$DOCKER_PASS" \
                      --namespace "${TARGET_NAMESPACE}" \
                      --dry-run=client \
                      -o yaml \
                      | kubectl apply -f -
                '''
            }
        }
    }

def deployEnvironment(String environmentName) {
    String releaseName = "exam-${environmentName}"

    withCredentials([
        file(
            credentialsId: 'config',
            variable: 'KUBECONFIG'
        )
    ]) {
        sh"""
            set -eu

            echo "Deploying image ${env.DOCKER_TAG}"
            echo "Environment: ${environmentName}"
            echo "Helm release: ${releaseName}"

            helm upgrade --install "${releaseName}" ./helm \
                      --namespace "${environmentName}" \
                      --set-string cast.tag="${env.DOCKER_TAG}" \
                      --set-string movie.tag="${env.DOCKER_TAG}" \
                      --wait \
                      --timeout 2m
            
            kubectl get pods \
                --namespace "${environmentName}"
        """
    }
}

def validateEnvironment(String environmentName) {
    withCredentials([
        file(
            credentialsId: 'config',
            variable: 'KUBECONFIG'
        )
    ]) {
        withEnv([
            "TARGET_NAMESPACE=${environmentName}"
        ]) {
            sh'''
                set -eu

                echo "Validating environment: $TARGET_NAMESPACE"

                echo "=== Kubernetes resources ==="

                kubectl get pods \
                  --namespace "$TARGET_NAMESPACE"

                kubectl get services \
                  --namespace "$TARGET_NAMESPACE"

                kubectl get deployments \
                  --namespace "$TARGET_NAMESPACE"

                kubectl get statefulsets \
                  --namespace "$TARGET_NAMESPACE"


                echo "=== Checking database StatefulSets ==="

                kubectl rollout status \
                  statefulset/cast-db \
                  --namespace "$TARGET_NAMESPACE" \
                  --timeout=180s

                kubectl rollout status \
                  statefulset/movie-db \
                  --namespace "$TARGET_NAMESPACE" \
                  --timeout=180s

                echo "=== Checking API Deployments ==="

                kubectl rollout status \
                  deployment/cast-service \
                  --namespace "$TARGET_NAMESPACE" \
                  --timeout=180s

                kubectl rollout status \
                  deployment/movie-service \
                  --namespace "$TARGET_NAMESPACE" \
                  --timeout=180s

                echo "=== Starting port-forwards ==="

                CAST_LOG="/tmp/cast-forward-${TARGET_NAMESPACE}-${BUILD_ID}.log"
                MOVIE_LOG="/tmp/movie-forward-${TARGET_NAMESPACE}-${BUILD_ID}.log"

                kubectl port-forward \
                  service/cast-service \
                  18002:8000 \
                  --namespace "$TARGET_NAMESPACE" \
                  > "$CAST_LOG" 2>&1 &

                CAST_PID=$!

                kubectl port-forward \
                  service/movie-service \
                  18001:8000 \
                  --namespace "$TARGET_NAMESPACE" \
                  > "$MOVIE_LOG" 2>&1 &

                MOVIE_PID=$!

                cleanup() {
                    echo "Stopping port-forwards"

                    kill "$CAST_PID" "$MOVIE_PID" \
                      2>/dev/null || true

                    wait "$CAST_PID" "$MOVIE_PID" \
                      2>/dev/null || true
                }

                trap cleanup EXIT

                echo "=== Waiting for port-forwards ==="

                ATTEMPT=1

                while [ "$ATTEMPT" -le 15 ]; do
                    if curl \
                        --silent \
                        --output /dev/null \
                        http://127.0.0.1:18002/health \
                    && curl \
                        --silent \
                        --output /dev/null \
                        http://127.0.0.1:18001/health
                    then
                        echo "Both APIs are reachable."
                        break
                    fi

                    if ! kill -0 "$CAST_PID" 2>/dev/null; then
                        echo "Cast port-forward failed:"
                        cat "$CAST_LOG"
                        exit 1
                    fi

                    if ! kill -0 "$MOVIE_PID" 2>/dev/null; then
                        echo "Movie port-forward failed:"
                        cat "$MOVIE_LOG"
                        exit 1
                    fi

                    if [ "$ATTEMPT" -eq 15 ]; then
                        echo "APIs did not become reachable."

                        echo "Cast port-forward log:"
                        cat "$CAST_LOG"

                        echo "Movie port-forward log:"
                        cat "$MOVIE_LOG"

                        exit 1
                    fi

                    ATTEMPT=$((ATTEMPT + 1))
                    sleep 2
                done

                echo "=== Testing Cast API ==="

                curl \
                  --fail \
                  --silent \
                  --show-error \
                  http://127.0.0.1:18002/health

                echo


                echo "=== Testing Movie API ==="

                curl \
                  --fail \
                  --silent \
                  --show-error \
                  http://127.0.0.1:18001/health

                echo

                echo "Environment $TARGET_NAMESPACE validated successfully."
                '''
            }
        }
    }

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
            steps {
                script {
                    prepareEnvironment('dev')
                }
            }
        }

        stage('Deploy DEV') {
            steps {
                script{
                    deployEnvironment('dev')
                }
            }
        }

        stage('Validate DEV') {
            steps {
                script {
                    validateEnvironment('dev')
                }
            }
        }
    }
}