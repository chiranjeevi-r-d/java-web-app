@Library('payment-shared-lib@main') _
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
            when {
                expression { maven_webapp.isStageEnabled('Checkout', params.STAGE_LIST) }
            }
            agent { label 'build-agent' }
            steps {
                script {
                    maven_webapp.checkout(
                        branch: params.GIT_BRANCH,
                        url: 'https://github.com/company/payment-app.git',
                        credentialsId: 'git-creds'
                    )
                }
            }
        }

        stage('Build') {
            when {
                expression { maven_webapp.isStageEnabled('Build', params.STAGE_LIST) }
            }
            agent { label 'build-agent' }
            steps {
                script {
                    maven_webapp.build(skipTests: true)
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { maven_webapp.isStageEnabled('SonarQube Analysis', params.STAGE_LIST) }
            }
            agent { label 'build-agent' }
            steps {
                script {
                    maven_webapp.sonarAnalysis(
                        sonarServer: 'SonarQube',
                        projectKey: 'payment-app'
                    )
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { maven_webapp.isStageEnabled('Quality Gate', params.STAGE_LIST) }
            }
            agent { label 'sonar' }
            steps {
                script {
                    maven_webapp.qualityGate(timeoutMinutes: 10, abortPipeline: true)
                }
            }
        }

        stage('Docker Build') {
            when {
                expression { maven_webapp.isStageEnabled('Docker Build', params.STAGE_LIST) }
            }
            agent { label 'docker-agent' }
            steps {
                script {
                    maven_webapp.dockerBuild(imageUri: env.ECR_URI, imageTag: env.IMAGE_TAG)
                }
            }
        }

        stage('Trivy Scan') {
            when {
                expression { maven_webapp.isStageEnabled('Trivy Scan', params.STAGE_LIST) }
            }
            agent { label 'docker-agent' }
            steps {
                script {
                    maven_webapp.trivyScan(imageUri: env.ECR_URI, imageTag: env.IMAGE_TAG)
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { maven_webapp.isStageEnabled('Push Docker Image', params.STAGE_LIST) }
            }
            agent { label 'docker-agent' }
            steps {
                script {
                    maven_webapp.pushDockerImage(
                        credentialsId: 'aws-creds',
                        awsRegion: env.AWS_REGION,
                        imageUri: env.ECR_URI,
                        imageTag: env.IMAGE_TAG
                    )
                }
            }
        }

        stage('Helm Package') {
            when {
                expression { maven_webapp.isStageEnabled('Helm Package', params.STAGE_LIST) }
            }
            agent { label 'helm-agent' }
            steps {
                script {
                    maven_webapp.helmPackage(chartPath: 'helm/payment')
                }
            }
        }

        stage('Push Helm Chart') {
            when {
                expression { maven_webapp.isStageEnabled('Push Helm Chart', params.STAGE_LIST) }
            }
            agent { label 'helm-agent' }
            steps {
                script {
                    maven_webapp.pushHelmChart(
                        credentialsId: 'aws-creds',
                        awsRegion: env.AWS_REGION,
                        helmRegistry: "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com",
                        chartFile: 'helm/payment-*.tgz'
                    )
                }
            }
        }

        stage('Deploy to EKS') {
            when {
                expression { maven_webapp.isStageEnabled('Deploy to EKS', params.STAGE_LIST) }
            }
            agent { label 'deploy-agent' }
            steps {
                script {
                    maven_webapp.deployToEks(
                        awsRegion: env.AWS_REGION,
                        eksCluster: env.EKS_CLUSTER,
                        releaseName: 'payment',
                        helmRegistry: "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/payment-charts",
                        namespace: 'payment',
                        imageUri: env.ECR_URI,
                        imageTag: env.IMAGE_TAG
                    )
                }
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
