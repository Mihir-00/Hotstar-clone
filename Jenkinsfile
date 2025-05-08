pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'mihir021/hotstar-clone'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        SONARQUBE_ENV = 'MySonarCloud' // Name configured in Jenkins global SonarQube servers
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
                sh 'docker version'
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Docker Scout - Image Scan') {
            when { expression { false } }
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
                   docker run --rm \
                      -v $PWD/zap-reports:/zap/wrk/:rw \
                      owasp/zap2docker-stable \
                      zap-baseline.py -t http://testphp.vulnweb.com -r zap-report.html
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
