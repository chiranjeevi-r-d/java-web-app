@Library('payment-shared-lib@main') _

mavenWebApp(
    gitBranch: 'main',
    environment: 'dev',
    stageList: 'Checkout,Build,SonarQube Analysis,Quality Gate,Docker Build,Trivy Scan,Push Docker Image,Helm Package,Push Helm Chart,Deploy to EKS',
    gitUrl: 'https://github.com/company/payment-app.git',
    credentialsId: 'git-creds',
    awsCredsId: 'aws-creds',
    awsRegion: 'ap-south-1',
    awsAccountId: '123456789012',
    ecrRepo: 'payment-app',
    helmRepo: 'payment-charts',
    eksCluster: 'prod-eks',
    projectKey: 'payment-app',
    sonarServer: 'SonarQube'
)
