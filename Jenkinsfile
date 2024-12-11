pipeline {
    agent {
        kubernetes {
            yaml """
${POD_TEMPLATE}
"""
        }
    }
   
    environment {
        // 환경변수들은 파이프라인 실행 시 전달받은 파라미터 사용
        NAMESPACE = "${params.NAMESPACE}"
        PROJECT_NAME = "${params.PROJECT_NAME}"
        DOCKER_USERNAME = "${params.DOCKER_USERNAME}"
        GIT_REPO = "${params.GIT_REPO}"
        GIT_BRANCH = "${params.GIT_BRANCH}"
        DOCKER_IMAGE = "${params.DOCKER_USERNAME}/${params.PROJECT_NAME}"
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
   
    stages {
        stage('Build Docker Image') {
            steps {
                container('kaniko') {
                    script {
                        sh """
                            /kaniko/executor \
                                --context . \
                                --destination ${DOCKER_IMAGE}:${DOCKER_TAG} \
                                --destination ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    script {
                        sh """
                            echo "Cleaning up existing deployment..."
                            kubectl delete deployment ${PROJECT_NAME} -n ${NAMESPACE} --ignore-not-found=true
                           
                            echo "Waiting for pods to terminate..."
                            kubectl wait --for=delete pod -l app=${PROJECT_NAME} -n ${NAMESPACE} --timeout=60s || true
                           
                            echo "Starting new deployment..."
                            sed -i 's|\${DOCKER_IMAGE}|${DOCKER_IMAGE}|g' k8s/deployment.yaml
                            sed -i 's|\${DOCKER_TAG}|${DOCKER_TAG}|g' k8s/deployment.yaml
                            kubectl apply -f k8s/deployment.yaml -n ${NAMESPACE}
                           
                            echo "Waiting for new deployment to be ready..."
                            kubectl rollout status deployment/${PROJECT_NAME} -n ${NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
