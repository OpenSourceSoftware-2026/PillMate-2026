pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'triplemone/pillmate-team'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.IMAGE_TAG = env.GIT_COMMIT
                }
                echo "branch: ${env.BRANCH_NAME}, tag: ${env.IMAGE_TAG}"
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t "$IMAGE_NAME:$IMAGE_TAG" .'
            }
        }

        stage('Docker Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG=$(mktemp -d)
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker push "$IMAGE_NAME:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            environment {
                GKE_PROJECT  = 'pillmate-project'
                GKE_CLUSTER  = 'pillmate-cluster'
                GKE_LOCATION = 'asia-northeast3-a'
                K8S_NS       = 'pillmate-team'
                KUBECONFIG   = "${WORKSPACE}/.kube/config"
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -e
                        gcloud container clusters get-credentials "$GKE_CLUSTER" \
                            --location "$GKE_LOCATION" --project "$GKE_PROJECT"

                        kubectl apply -f k8s/namespace.yaml

                        set +x
                        kubectl -n "$K8S_NS" create secret docker-registry dockerhub-pull-secret \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username="$DOCKERHUB_USER" \
                            --docker-password="$DOCKERHUB_TOKEN" \
                            --dry-run=client -o yaml | kubectl apply -f -
                        set -x

                        kubectl apply -f k8s/service.yaml
                        sed "s|IMAGE_TAG|${IMAGE_TAG}|g" k8s/deployment.yaml | kubectl apply -f -
                        kubectl -n "$K8S_NS" annotate deployment/pillmate \
                            kubernetes.io/change-cause="build #${BUILD_NUMBER} ${IMAGE_NAME}:${IMAGE_TAG}" --overwrite

                        kubectl -n "$K8S_NS" rollout status deployment/pillmate --timeout=300s
                        kubectl -n "$K8S_NS" get pods,svc -o wide
                    '''
                }
            }
            post {
                failure {
                    sh 'KUBECONFIG="$WORKSPACE/.kube/config" kubectl -n pillmate-team rollout undo deployment/pillmate || true'
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}
