pipeline {
    agent any
    tools {
        maven 'Maven-3.9.9'
    }

    environment {
        AWS_REGION   = "ap-south-1"
        ECR_REGISTRY = "623595292733.dkr.ecr.ap-south-1.amazonaws.com"
        ECR_REPO     = "workshop/kubernetes"
        IMAGE_TAG    = "latest"
        EKS_CLUSTER  = "eks-kub-workshop-ap-001"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                userRemoteConfigs: [[url: 'https://github.com/Sudhanshuoberoi/Boardgame.git',
                credentialsId: '7cc1e256-c100-4f13-8d25-7950c76b0de7']]])
            }
        }

        stage('Code Analysis') {
            environment {
                scannerHome = tool 'sonarqube'  
            }
            steps {
                script {
                    withSonarQubeEnv('sonarqube') {
                        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_TOKEN')]) {
                            sh """
                                mvn clean install sonar:sonar \
                                  -DskipTests \
                                  -Dsonar.projectKey=boardgame-app \
                                  -Dsonar.projectName="Boardgame Project" \
                                  -Dsonar.projectVersion=${BUILD_NUMBER} \
                                  -Dsonar.sources=src/main/java \
                                  -Dsonar.login=$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn -v'
                sh 'mvn clean package -Dmaven.compiler.plugin.version=3.13.0'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                  docker build -t $ECR_REGISTRY/$ECR_REPO:$BUILD_NUMBER .
                """
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh '''
                   trivy image --format json --output trivy-report-$BUILD_NUMBER.json $ECR_REGISTRY/$ECR_REPO:$BUILD_NUMBER
                '''
            }
        }
        
        stage('Convert report to CSV') {
            steps {
                sh '''
                   jq -r '.Results[]?.Vulnerabilities[]? | [
                        .VulnerabilityID,
                        .PkgName,
                        .InstalledVersion,
                        .FixedVersion,
                        .Severity,
                        .Title
                    ] | @csv' trivy-report-$BUILD_NUMBER.json > trivy-report-$BUILD_NUMBER.csv
                '''
            }
            post {
                success {
                    emailext (
                        subject: "Trivy Scan Report - Build #${BUILD_NUMBER}",
                        body: "Attached is the Trivy vulnerability scan report for build #${BUILD_NUMBER}.",
                        to: "sudhanshuoberoi110@gmail.com",
                        attachmentsPattern: "trivy-report-${BUILD_NUMBER}.csv"
                    )
                }
            }
        }

        stage('ECR Login') {
            steps {
                withAWS(region: "${AWS_REGION}") {
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push $ECR_REGISTRY/$ECR_REPO:$BUILD_NUMBER'
            }
        }

        stage('Update Image Tag') {
            steps {
                script {
                    sh '''
                       sed -e "s|ECR_REGISTRY|$ECR_REGISTRY|g" \
                           -e "s|ECR_REPO|$IMAGE_TECR_REPO|g" \
                           -e "s|BUILD_ID|$BUILD_NUMBER|g" \
                           deployment-service.yaml > deployment.yaml
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Update kubeconfig
                    sh 'cat deployment.yaml'
                    sh "aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER"
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }
    post {
        success {
            emailext(
                to: 'sudhanshuoberoi110@gmail.com',
                subject: "✅ Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Succeeded",
                body: "Good news! The job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' completed successfully.\n\nCheck console: ${env.BUILD_URL}"
            )
        }
        failure {
            emailext(
                to: 'sudhanshuoberoi110@gmail.com',
                subject: "❌ Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Failed",
                body: "The job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has failed.\n\nCheck console: ${env.BUILD_URL}"
            )
        }
    }
}
