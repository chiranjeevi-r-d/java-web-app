pipeline {
    agent none
    tools {
        maven 'Maven-3.9.0'
        jdk 'JDK-21'
    }
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'qa', 'uat', 'prod'], description: 'Deployment Environment')
        string(
            name: 'STAGE_LIST',
            defaultValue: 'Checkout,Build,SonarQube Analysis,Quality Gate,Docker Build,Trivy Scan,Push Docker Image,Helm Package,Push Helm Chart,Deploy to EKS',
            description: 'Comma-separated list of stages to run'
        )
    }
    environment {
        AWS_REGION     = "ap-south-1"
        AWS_ACCOUNT_ID = "123456789012"
        ECR_REPO       = "payment-app"
        ECR_URI        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        HELM_REPO      = "payment-charts"
        EKS_CLUSTER    = "prod-eks"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        EMAIL_TO       = "devops-team@company.com"
    }
    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: params.GIT_BRANCH,
                    credentialsId: 'git-creds',
                    url: 'https://github.com/company/payment-app.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=payment-app'
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh "docker build -t ${env.ECR_URI}:${env.IMAGE_TAG} ."
            }
        }
        stage('Trivy Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${env.ECR_URI}:${env.IMAGE_TAG}"
            }
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        aws ecr get-login-password --region ${env.AWS_REGION} | \
                        docker login --username AWS --password-stdin ${env.ECR_URI}
                        docker push ${env.ECR_URI}:${env.IMAGE_TAG}
                    """
                }
            }
        }
        stage('Helm Package & Push Helm Chart') {
            steps {
                sh """
                    helm dependency update helm/payment
                    helm package helm/payment
                    aws ecr get-login-password --region ${env.AWS_REGION} | \
                    helm registry login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com
                    helm push *.tgz oci://${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/
                """
            }
        }
        stage('Deploy to EKS') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${env.AWS_REGION} --name ${env.EKS_CLUSTER}
                    helm upgrade --install payment \
                        oci://${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/payment-charts \
                        --namespace payment --create-namespace \
                        --set image.repository=${env.ECR_URI} \
                        --set image.tag=${env.IMAGE_TAG}
                """
            }
        }
    }
    post {
        success {
            emailext(
                subject: "SUCCESS: ${JOB_NAME}-${BUILD_NUMBER}",
                body: """Build Successful
                Job: ${JOB_NAME}
                Build: ${BUILD_NUMBER}
                Image: ${ECR_URI}:${IMAGE_TAG}""",
                to: "${EMAIL_TO}"
            )
        }
        failure {
            emailext(
                subject: "FAILED: ${JOB_NAME}-${BUILD_NUMBER}",
                body: "Please check Jenkins logs",
                to: "${EMAIL_TO}"
            )
        }
    }
}
