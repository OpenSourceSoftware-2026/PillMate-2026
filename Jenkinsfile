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
            steps {
                // GKE 배포는 5번 작업 완료 후 추가 예정
                echo "deploy target image: ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}
