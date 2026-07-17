pipeline {

    agent any

    tools {
        maven 'maven3'      // Name must match: Manage Jenkins -> Tools -> Maven installations
        jdk    'jdk21'      // Name must match: Manage Jenkins -> Tools -> JDK installations
        nodejs 'node20'     // Name must match: Manage Jenkins -> Tools -> NodeJS installations
    }

    environment {
        DOCKERHUB_CREDS   = credentials('dockerhub-creds')        // Username + Password/Token credential
        DOCKER_NAMESPACE  = 'basavabm'                             // Docker Hub username / org
        BACKEND_IMAGE     = "${DOCKER_NAMESPACE}/sim-backend"
        FRONTEND_IMAGE    = "${DOCKER_NAMESPACE}/sim-frontend"
        IMAGE_TAG         = "${env.BUILD_NUMBER}"
        KUBECONFIG_CRED   = credentials('kubeconfig-file')         // Secret file credential (kubeconfig)
        K8S_NAMESPACE     = 'smartims'
        SONAR_PROJECT_KEY = 'smart-incident-management'
        SLACK_CHANNEL     = '#ci-cd-alerts'
        EMAIL_RECIPIENTS  = 'basavabm30@gmail.com'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '15'))
        disableConcurrentBuilds()
    }

    stages {

        stage('1. Code Checkout from SCM') {
            steps {
                checkout scm
            }
        }

        stage('2. Build the Application') {
            parallel {
                stage('Build Backend (Maven)') {
                    steps {
                        dir('backend') {
                            sh 'mvn -B clean package -DskipTests'
                        }
                    }
                }
                stage('Build Frontend (npm)') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                            sh 'npm run build'
                        }
                    }
                }
            }
        }

        stage('3. Run Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh 'mvn -B test'
                        }
                    }
                    post {
                        always {
                            junit testResults: 'backend/target/surefire-reports/*.xml', allowEmptyResults: true
                        }
                    }
                }
                stage('Frontend Lint (no test script defined yet)') {
                    steps {
                        dir('frontend') {
                            sh 'npm run lint'
                        }
                    }
                }
            }
        }

        stage('4. Perform Code Quality Check (SonarQube)') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('sonar-server') {   // Name must match: Manage Jenkins -> System -> SonarQube servers
                        sh """
                            mvn -B sonar:sonar \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY}-backend \
                              -Dsonar.projectName=smart-incident-management-backend
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Build the Docker Image') {
            parallel {
                stage('Backend Image') {
                    steps {
                        dir('backend') {
                            sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ."
                        }
                    }
                }
                stage('Frontend Image') {
                    steps {
                        dir('frontend') {
                            sh "docker build --build-arg VITE_API_URL=${VITE_API_URL ?: 'http://35.200.195.146:8081'} -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ."
                        }
                    }
                }
            }
        }

        stage('6. Tag the Docker Image') {
            steps {
                sh """
                    docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                    docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                """
            }
        }

        stage('7. Push the Docker Image to Docker Hub') {
            steps {
                sh "echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin"
                sh """
                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest
                """
            }
        }

        stage('8. Deploy the Application to Kubernetes') {
            steps {
                withEnv(["KUBECONFIG=${KUBECONFIG_CRED}"]) {
                    sh """
                        kubectl apply -f k8s/00-namespace.yaml
                        kubectl apply -f k8s/02-postgres.yaml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/backend-deployment.yaml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/frontend-deployment.yaml -n ${K8S_NAMESPACE}

                        kubectl set image deployment/sim-backend sim-backend=${BACKEND_IMAGE}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
                        kubectl set image deployment/sim-frontend sim-frontend=${FRONTEND_IMAGE}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
                    """
                }
                // NOTE: k8s/01-secret.yaml is intentionally NOT applied here because it currently
                // contains plaintext credentials committed to the repo. Create/update that secret
                // out-of-band (e.g. `kubectl create secret ... --from-literal=...` using Jenkins
                // credentials) instead of applying the committed file.
            }
        }

        stage('9. Verify the Deployment') {
            steps {
                withEnv(["KUBECONFIG=${KUBECONFIG_CRED}"]) {
                    sh """
                        kubectl rollout status deployment/sim-backend -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl rollout status deployment/sim-frontend -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }

        success {
            emailext(
                subject: "SUCCESS: Build #${env.BUILD_NUMBER} - ${env.JOB_NAME}",
                body: """Build #${env.BUILD_NUMBER} of ${env.JOB_NAME} succeeded.

Backend image:  ${BACKEND_IMAGE}:${IMAGE_TAG}
Frontend image: ${FRONTEND_IMAGE}:${IMAGE_TAG}

View console output: ${env.BUILD_URL}console""",
                to: "${EMAIL_RECIPIENTS}"
            )
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'good',
                message: "✅ *${env.JOB_NAME}* build #${env.BUILD_NUMBER} succeeded and deployed to *${K8S_NAMESPACE}*.\n${env.BUILD_URL}"
            )
        }

        failure {
            emailext(
                subject: "FAILED: Build #${env.BUILD_NUMBER} - ${env.JOB_NAME}",
                body: """Build #${env.BUILD_NUMBER} of ${env.JOB_NAME} failed.

Check console output: ${env.BUILD_URL}console""",
                to: "${EMAIL_RECIPIENTS}"
            )
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'danger',
                message: "❌ *${env.JOB_NAME}* build #${env.BUILD_NUMBER} failed.\n${env.BUILD_URL}console"
            )
        }
    }
}
