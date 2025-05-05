pipeline {
    agent any
    tools {
        sonarQubeScanner 'Default'
    }

    environment {
        DOCKER_IMAGE = 'mihir021/Hotstar-clone'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        SONARQUBE_ENV = 'MySonarQube' // Name configured in Jenkins global SonarQube servers
        SONAR_TOKEN = credentials('sonarcloud-token')
        AWS_REGION = 'eu-north-1'
        CLUSTER_NAME = 'your-eks-cluster-name'
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
                    withSonarQubeEnv('MySonarCloud') {
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

        stage('Docker Scout - Image Scan') {
            steps {
                sh '''
                    docker scout quickview ${DOCKER_IMAGE}:${BUILD_NUMBER} || echo "Scout scan warnings or issues detected"
                '''
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        dockerImage.push()
                        dockerImage.push('latest') // Optional
                    }
                }
            }
        }

        stage('Update kubeconfig for EKS') {
            when { expression { false } }
            steps {
                sh '''
                    aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
                '''
            }
        }

        stage('Deploy to EKS') {
            when { expression { false } }
            steps {
                sh '''
                    kubectl apply -f k8s/deployment.yaml
                    kubectl rollout status deployment your-deployment-name
                '''
            }
        }

        stage('OWASP ZAP - Dynamic Security Test') {
            steps {
                sh '''
                    docker run --rm -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py \
                        -t http://testphp.vulnweb.com \
                        -r zap-report.html || echo "ZAP scan completed with warnings"
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            }
        }
    }
}
