pipeline {
    agent any

    environment {
        SONAR_URL = 'http://localhost:9000'

        NEXUS_URL = 'http://localhost:8081/repository/java-repo/'
        NEXUS_SETTINGS_ID = 'd2093c9d-1921-46ce-bc0e-5c19c03900ad'

        AWS_ACCOUNT_ID = 'YOUR_ACCOUNT_ID'
        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'YOUR_ECR_REPOSITORY'
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=spring-boot-demo \
                          -Dsonar.projectName=spring-boot-demo \
                          -Dsonar.host.url=$SONAR_URL \
                          -Dsonar.token=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Nexus Upload') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    sh '''
                        mvn deploy -DskipTests \
                        -DaltDeploymentRepository=nexus::default::http://localhost:8081/repository/java-repo/
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY/java-demo-app:$BUILD_NUMBER .
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY/java-demo-app:$BUILD_NUMBER
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-cred'
                ]]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin \
                        $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                        docker push \
                        $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY/java-demo-app:$BUILD_NUMBER
                    '''
                }
            }
        }
    }

    post {
        failure {
            mail to: 'adelelnimrwhats@example.com',
                 subject: "Pipeline Failed: ${currentBuild.fullDisplayName}",
                 body: "Build failed. Check console output at ${env.BUILD_URL}"
        }
    }
}