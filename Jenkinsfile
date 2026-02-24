pipeline {
    agent any

    environment {
        APP_NAME      = 'jenkins-k8s-demo'
        BUILD_TAG     = "v${BUILD_NUMBER}"
        MINIKUBE_HOME = '/var/lib/jenkins'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Install & Test') {
            steps {
                sh 'npm install'
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building image: ${APP_NAME}:${BUILD_TAG}"
                sh """
                    export MINIKUBE_HOME=/var/lib/jenkins
                    minikube image build -t ${APP_NAME}:${BUILD_TAG} .
                    minikube image tag ${APP_NAME}:${BUILD_TAG} ${APP_NAME}:latest
                """
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${APP_NAME}:${BUILD_TAG}
                    kubectl set env deployment/${APP_NAME} \
                        BUILD_NUMBER=${BUILD_NUMBER}
                    kubectl rollout status deployment/${APP_NAME} --timeout=60s
                """
            }
        }

        stage('Verify') {
            steps {
                sh """
                    echo "--- Pods ---"
                    kubectl get pods -l app=${APP_NAME}
                    echo "--- Service ---"
                    kubectl get svc
                    echo "--- App URL ---"
                    minikube service ${APP_NAME}-service --url
                """
            }
        }
    }

    post {
        success {
            echo "Build #${BUILD_NUMBER} deployed to Minikube successfully!"
            echo "App URL: http://172.22.117.176:9090"
            sh "kubectl port-forward service/jenkins-k8s-demo-service 9090:80 --address=0.0.0.0 &"
        }
        failure {
            echo "Failed! Rolling back..."
            sh "kubectl rollout undo deployment/${APP_NAME} || true"
        }
        always {
            cleanWs()
        }
    }
}