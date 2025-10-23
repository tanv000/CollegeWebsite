pipeline {
    agent any

    environment {
        IMAGE_NAME = "vjagvi/college-website"
        ECR_REPO   = "708972351530.dkr.ecr.ap-south-1.amazonaws.com/college_website"
        REGION     = "ap-south-1"
        AWS_CLI    = "C:\\Program Files\\Amazon\\AWSCLI\\bin\\aws.exe"
        TERRAFORM  = "C:\\Terraform\\terraform.exe"
    }

    stages {
        stage('Clone Repository') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/tanv000/CollegeWebsite.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                bat 'docker build -t %IMAGE_NAME%:latest .'
            }
        }

        stage('Push to AWS ECR') {
            steps {
                echo 'Pushing image to AWS ECR...'
                withCredentials([usernamePassword(credentialsId: 'AWS_ECR_CREDENTIALS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    bat """
                    set AWS_ACCESS_KEY_ID=%AWS_ACCESS_KEY_ID%
                    set AWS_SECRET_ACCESS_KEY=%AWS_SECRET_ACCESS_KEY%
                    "%AWS_CLI%" ecr get-login-password --region %REGION% | docker login --username AWS --password-stdin %ECR_REPO%
                    docker tag %IMAGE_NAME%:latest %ECR_REPO%:latest
                    docker push %ECR_REPO%:latest
                    """
                }
            }
        }

        stage('Deploy with Terraform') {
            steps {
                echo 'Deploying EC2 instance and running Docker container...'
                withCredentials([usernamePassword(credentialsId: 'AWS_ECR_CREDENTIALS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('terraform') {
                        bat """
                        set AWS_ACCESS_KEY_ID=%AWS_ACCESS_KEY_ID%
                        set AWS_SECRET_ACCESS_KEY=%AWS_SECRET_ACCESS_KEY%
                        "%TERRAFORM%" init
                        "%TERRAFORM%" apply -auto-approve
                        """
                    }
                }
            }
        }

        stage('Fetch EC2 Public IP') {
            steps {
                echo 'Fetching EC2 public IP...'
                withCredentials([usernamePassword(credentialsId: 'AWS_ECR_CREDENTIALS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('terraform') {
                        script {
                            def ec2_ip = bat(script: """
                                set AWS_ACCESS_KEY_ID=%AWS_ACCESS_KEY_ID%
                                set AWS_SECRET_ACCESS_KEY=%AWS_SECRET_ACCESS_KEY%
                                "%AWS_CLI%" ec2 describe-instances --region %REGION% --filters "Name=tag:Name,Values=CollegeWebsite-EC2" "Name=instance-state-name,Values=running" --query "Reservations[*].Instances[*].PublicIpAddress" --output text
                                """, returnStdout: true).trim()
                            
                            echo "EC2 Public IP: ${ec2_ip}"
                            echo "Visit your website at: http://${ec2_ip}"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful. Your College Website is live!'
        }
        failure {
            echo 'Build or deployment failed!'
        }
    }
}
