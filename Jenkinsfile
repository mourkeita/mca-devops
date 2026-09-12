pipeline {

    agent any

    parameters {
        string(
            name: 'AWS_REGION',
            defaultValue: 'eu-west-3',
            description: 'AWS region'
        )

        string(
            name: 'EKS_CLUSTER',
            defaultValue: 'mca-devops-cluster',
            description: 'EKS cluster name'
        )

        string(
            name: 'DOCKER_IMAGE_PREFIX',
            defaultValue: 'mkeita',
            description: 'Docker Hub username'
        )
    }

    environment {

        FRONTEND_IMAGE = "${params.DOCKER_IMAGE_PREFIX}/mca-devops-frontend"
        BACKEND_IMAGE  = "${params.DOCKER_IMAGE_PREFIX}/mca-devops-backend"

        IMAGE_TAG = "${BUILD_NUMBER}"

        K8S_NAMESPACE = "mca"

        AWS_DEFAULT_REGION = "${params.AWS_REGION}"
    }

    stages {

        /*
         * ==========================================
         * 1. CHECKOUT
         * ==========================================
         */
        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "Git commit:"
                    git rev-parse --short HEAD
                '''
            }
        }


        /*
         * ==========================================
         * 2. BUILD FRONTEND
         * ==========================================
         */
        stage('Build Frontend') {
            steps {
                dir('frontend') {

                    sh '''
                        echo "Installing frontend dependencies..."

                        npm ci

                        echo "Building Angular application..."

                        npm run build
                    '''
                }
            }
        }


        /*
         * ==========================================
         * 3. BUILD BACKEND
         * ==========================================
         */
        stage('Build Backend') {
            steps {
                dir('backend') {

                    sh '''
                        echo "Building backend..."

                        mvn clean package -DskipTests
                    '''
                }
            }
        }


        /*
         * ==========================================
         * 4. DOCKER BUILD FRONTEND
         * ==========================================
         */
        stage('Docker Build Frontend') {
            steps {

                sh '''
                    echo "Building frontend Docker image..."

                    docker build \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        ./frontend
                '''
            }
        }


        /*
         * ==========================================
         * 5. DOCKER BUILD BACKEND
         * ==========================================
         */
        stage('Docker Build Backend') {
            steps {

                sh '''
                    echo "Building backend Docker image..."

                    docker build \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        ./backend
                '''
            }
        }


        /*
         * ==========================================
         * 6. PUSH DOCKER IMAGES
         * ==========================================
         */
        stage('Push Docker Images') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "Logging into Docker Hub..."

                        echo "${DOCKER_PASSWORD}" | \
                        docker login \
                        -u "${DOCKER_USERNAME}" \
                        --password-stdin


                        echo "Pushing frontend..."

                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest


                        echo "Pushing backend..."

                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest


                        docker logout
                    '''
                }
            }
        }


        /*
         * ==========================================
         * 7. CREATE EKS CLUSTER
         * ==========================================
         */
        stage('Create EKS Infrastructure') {

            steps {

                sh '''
                    echo "Checking EKS cluster..."

                    if aws eks describe-cluster \
                        --name ${EKS_CLUSTER} \
                        --region ${AWS_DEFAULT_REGION} \
                        >/dev/null 2>&1
                    then

                        echo "EKS cluster already exists."

                    else

                        echo "Creating EKS cluster..."

                        eksctl create cluster \
                            -f eks/cluster.yaml

                    fi
                '''
            }
        }


        /*
         * ==========================================
         * 8. CONFIGURE KUBECTL
         * ==========================================
         */
        stage('Configure kubectl') {

            steps {

                sh '''
                    aws eks update-kubeconfig \
                        --region ${AWS_DEFAULT_REGION} \
                        --name ${EKS_CLUSTER}

                    kubectl get nodes
                '''
            }
        }


        /*
         * ==========================================
         * 9. CREATE NAMESPACE
         * ==========================================
         */
        stage('Create Namespace') {

            steps {

                sh '''
                    kubectl create namespace ${K8S_NAMESPACE} \
                        --dry-run=client \
                        -o yaml | kubectl apply -f -
                '''
            }
        }


        /*
         * ==========================================
         * 10. DEPLOY POSTGRESQL
         * ==========================================
         */
        stage('Deploy PostgreSQL') {

            steps {

                sh '''
                    kubectl apply \
                        -f k8s/secret.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl apply \
                        -f k8s/postgres-pvc.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl apply \
                        -f k8s/postgres-deployment.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl apply \
                        -f k8s/postgres-service.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl rollout status \
                        deployment/postgres \
                        -n ${K8S_NAMESPACE} \
                        --timeout=180s
                '''
            }
        }


        /*
         * ==========================================
         * 11. DEPLOY BACKEND
         * ==========================================
         */
        stage('Deploy Backend') {

            steps {

                sh '''
                    kubectl apply \
                        -f k8s/backend-deployment.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl apply \
                        -f k8s/backend-service.yaml \
                        -n ${K8S_NAMESPACE}


                    kubectl set image \
                        deployment/backend \
                        backend=${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -n ${K8S_NAMESPACE}


                    kubectl rollout status \
                        deployment/backend \
                        -n ${K8S_NAMESPACE} \
                        --timeout=180s
                '''
            }
        }


        /*
         * ==========================================
         * 12. DEPLOY FRONTEND
         * ==========================================
         */
        stage('Deploy Frontend') {

            steps {

                sh '''
                    kubectl apply \
                        -f k8s/frontend-deployment.yaml \
                        -n ${K8S_NAMESPACE}

                    kubectl apply \
                        -f k8s/frontend-service.yaml \
                        -n ${K8S_NAMESPACE}


                    kubectl set image \
                        deployment/frontend \
                        frontend=${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -n ${K8S_NAMESPACE}


                    kubectl rollout status \
                        deployment/frontend \
                        -n ${K8S_NAMESPACE} \
                        --timeout=180s
                '''
            }
        }


        /*
         * ==========================================
         * 13. DEPLOY INGRESS
         * ==========================================
         */
        stage('Deploy Ingress') {

            steps {

                sh '''
                    kubectl apply \
                        -f k8s/ingress.yaml \
                        -n ${K8S_NAMESPACE}
                '''
            }
        }


        /*
         * ==========================================
         * 14. VERIFY
         * ==========================================
         */
        stage('Verify Deployment') {

            steps {

                sh '''
                    echo "=============================="
                    echo "PODS"
                    echo "=============================="

                    kubectl get pods \
                        -n ${K8S_NAMESPACE}


                    echo "=============================="
                    echo "SERVICES"
                    echo "=============================="

                    kubectl get svc \
                        -n ${K8S_NAMESPACE}


                    echo "=============================="
                    echo "INGRESS"
                    echo "=============================="

                    kubectl get ingress \
                        -n ${K8S_NAMESPACE}
                '''
            }
        }
    }


    /*
     * ==========================================
     * POST ACTIONS
     * ==========================================
     */
    post {

        success {
            echo "======================================"
            echo "DEPLOYMENT SUCCESSFUL"
            echo "Build: ${BUILD_NUMBER}"
            echo "Frontend: ${FRONTEND_IMAGE}:${IMAGE_TAG}"
            echo "Backend:  ${BACKEND_IMAGE}:${IMAGE_TAG}"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "DEPLOYMENT FAILED"
            echo "======================================"
        }
    }
}

