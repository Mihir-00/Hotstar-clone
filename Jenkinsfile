pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'mihir021/hotstar-clone'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        SONARQUBE_ENV = 'MySonarCloud' // Name configured in Jenkins global SonarQube servers
        SONAR_TOKEN = credentials('sonarcloud-token')
        AWS_REGION = 'eu-north-1'
        CLUSTER_NAME = ''
        SERVICE_NAME = 'hotstar-service'
        NAMESPACE = 'default'
        APP_URL = ''
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube - Static Code Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=mihir-devops_hotstarclone \
                              -Dsonar.organization=mihir-devops \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=https://sonarcloud.io \
                              -Dsonar.login=$SONAR_TOKEN \
                              -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                        '''
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Update kubeconfig for EKS') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                  script {
                    CLUSTER_NAME = sh(script: 'terraform output -raw cluster_name', returnStdout: true).trim()
                    env.CLUSTER_NAME = CLUSTER_NAME
                }
                sh '''
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl apply -f k8s/deployment.yaml
                '''
            }
        }
        stage('Get LoadBalancer URL') {
          steps {
              script {
                  sh '''
                    echo "Waiting for LoadBalancer to be ready..."
                    for i in {1..30}; do
                        HOST=$(kubectl get svc ${SERVICE_NAME} -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                        if [ ! -z "$HOST" ]; then
                            echo "APP_URL=http://$HOST" > app_url.txt
                            break
                        fi
                        sleep 10
                    done
                '''
                APP_URL = sh(script: "cat app_url.txt | cut -d '=' -f2", returnStdout: true).trim()
                echo "Application URL is: ${APP_URL}"
                }
            }
        }

        stage('OWASP ZAP - Dynamic Security Test') {
            steps {
                sh '''
                    docker pull zaproxy/zap-stable
                    CONTAINER_ID=$(docker run -d --user root \
                    -v ${WORKSPACE}:/zap/wrk:rw \
                    zaproxy/zap-stable zap-baseline.py -t ${APP_URL} -r report.html -I -d)
                    # Wait for scan to complete
                    docker logs -f "$CONTAINER_ID"
                    mkdir -p ${WORKSPACE}/zap_report
                    docker cp "$CONTAINER_ID":/zap/wrk/report.html ${WORKSPACE}/zap_report/report.html
                    docker rm "$CONTAINER_ID"
                '''
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap_report/report.html', allowEmptyArchive: false
        }
    }

}
