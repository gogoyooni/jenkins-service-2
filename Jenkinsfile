pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            label 'kubeagent'  // Jenkins UI에서 설정한 label과 일치
            namespace 'devops-tools'  // Jenkins agent가 실행될 네임스페이스
            serviceAccount 'jenkins-admin'  // 우리가 설정한 ServiceAccount
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                  namespace: devops-tools
                  labels:
                    jenkins: agent
                    component: kubeagent
                spec:
                  serviceAccountName: jenkins-admin
                  containers:
                  - name: kaniko
                    image: gcr.io/kaniko-project/executor:debug
                    command:
                    - /busybox/cat
                    tty: true
                    volumeMounts:
                    - name: docker-config
                      mountPath: /kaniko/.docker/
                  - name: kubectl
                    image: bitnami/kubectl:latest
                    command:
                    - sleep
                    args:
                    - 99d
                    tty: true
                  volumes:
                  - name: docker-config
                    secret:
                      secretName: docker-credentials
                      items:
                        - key: .dockerconfigjson
                          path: config.json
            '''
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
        // ... stages 내용은 동일 ...
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