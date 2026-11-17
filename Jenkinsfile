
pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-2'
        AWS_ACCOUNT_ID = '651821641416'
        ECR_REPOSITORY = 'employee-portal'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        IMAGE = "${IMAGE_NAME}:${BUILD_NUMBER}"

        K8S_NAMESPACE = 'default'
    }

    stages {

        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                    echo "Checking application files..."

                    ls -l

                    test -f index.html
                    test -f Dockerfile
                    test -f Jenkinsfile
                    test -f k8s/deployment.yaml
                    test -f k8s/service.yaml

                    echo "Application files verified successfully"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build -t $IMAGE .

                    docker tag $IMAGE $IMAGE_NAME:latest

                    echo "Docker image built successfully"
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    echo "Starting smoke test..."

                    docker rm -f smoke-test || true

                    docker run -d \
                        --name smoke-test \
                        -p 8090:80 \
                        $IMAGE

                    sleep 5

                    curl -f http://localhost:8090

                    echo "Smoke test passed successfully"
                '''
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh '''
                    echo "Logging in to Amazon ECR..."

                    aws ecr get-login-password \
                        --region $AWS_REGION | \
                        docker login \
                        --username AWS \
                        --password-stdin $ECR_REGISTRY

                    echo "Pushing Docker image to ECR..."

                    docker push $IMAGE

                    docker push $IMAGE_NAME:latest

                    docker logout $ECR_REGISTRY

                    echo "Docker image pushed successfully"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "Checking Kubernetes connection..."

                    kubectl cluster-info

                    echo "Creating ECR authentication secret..."

                    ECR_PASSWORD=$(aws ecr get-login-password \
                        --region $AWS_REGION)

                    kubectl create secret docker-registry \
                        ecr-registry-secret \
                        --docker-server=$ECR_REGISTRY \
                        --docker-username=AWS \
                        --docker-password="$ECR_PASSWORD" \
                        --namespace=$K8S_NAMESPACE \
                        --dry-run=client -o yaml | kubectl apply -f -

                    unset ECR_PASSWORD

                    echo "Deploying application to Kubernetes..."

                    sed "s|IMAGE_PLACEHOLDER|$IMAGE|g" \
                        k8s/deployment.yaml | kubectl apply -n $K8S_NAMESPACE -f -

                    kubectl apply -f k8s/service.yaml \
                        --namespace=$K8S_NAMESPACE

                    echo "Kubernetes deployment applied successfully"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Waiting for Kubernetes deployment..."

                    kubectl rollout status \
                        deployment/employee-portal \
                        --namespace=$K8S_NAMESPACE \
                        --timeout=120s

                    echo "Checking pods..."

                    kubectl get pods -n $K8S_NAMESPACE

                    echo "Checking deployment..."

                    kubectl get deployment -n $K8S_NAMESPACE

                    echo "Checking services..."

                    kubectl get svc -n $K8S_NAMESPACE

                    echo "Deployment verification completed"
                '''
            }
        }
    }

    post {
        always {
            echo 'Cleaning up smoke test container...'

            sh 'docker rm -f smoke-test || true'
        }

        success {
            echo 'CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check the Jenkins console output.'
        }
    }
}

