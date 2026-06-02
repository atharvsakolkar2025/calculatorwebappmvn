pipeline {
    agent { label 'linux' } // Use a Jenkins agent with Docker capabilities
    tools {
        // terraform 'Terraform-1.5.7'   // Must match Global Tool Configuration
        maven 'my-maven'             // Your Maven installation
    }
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Select environment to deploy')
        booleanParam(name: 'APPLY', defaultValue: true, description: 'Run terraform apply?')
        booleanParam(name: 'DESTROY', defaultValue: false, description: 'Run terraform destroy?')
    }
    environment {
        my_aws_access = credentials('AWS-CRED')
        IMAGE_NAME = "calcwebappmvn:v1"
        cluster_name = "jenkins-cluster"
    }
    stages {

        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/atharvsakolkar2025/calculatorwebappmvn'
                echo "Code Checked-out Successfully!!"
                sh 'ls -la'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        try {
                            def qg = waitForQualityGate()
                            echo "Quality Gate Status: ${qg.status}"
                            if (qg.status != 'OK') {
                                error "Quality Gate failed: ${qg.status}"
                            }
                        } catch (Exception e) {
                            echo "Quality Gate check failed: ${e.message}"
                            error "Quality Gate stage failed"
                        }
                    }
                }
            }
        }

        stage('Package Application .war') {
            steps {
                sh 'mvn clean package'
                echo "Maven Package Goal Executed Successfully!"
                sh 'ls -la'
            }
        }

        stage('docker image build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
                echo "Docker Image Built Successfully!!"
                sh 'docker images'
            }
        }

        stage('ECRLogin') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 193619625891.dkr.ecr.us-east-1.amazonaws.com'
                echo "Logged in to AWS ECR Successfully!!"

                sh 'docker tag ${IMAGE_NAME} 193619625891.dkr.ecr.us-east-1.amazonaws.com/atharv:${BUILD_NUMBER}'
                echo "Docker Image Tagged Successfully!!"
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push 193619625891.dkr.ecr.us-east-1.amazonaws.com/atharv:${BUILD_NUMBER}'
                echo "Docker Image Pushed to ECR Successfully!!"
            }
        }

        
        stage('kubeconfig setup') {
            steps {
                sh 'aws eks update-kubeconfig --region us-east-1 --name jenkins-cluster'
                echo "Kubeconfig setup completed successfully!!"
            }
        }

        stage('get all resources') {
            steps {
                sh 'kubectl get all'
                echo "Verified access to EKS cluster successfully!!"
            }
        }
    }
}

  stage('deploy to eks') {
            steps {

                sh 'kubectl apply -f calc-deployment-svc.yaml'
                sh 'kubectl get all'
                sh 'sleep 20'
                sh 'kubectl get svc -o wide'
                echo ".war application deployed to EKS cluster successfully!!"

                //sh 'kubectl apply -f k8s-deployment.yaml'
                //echo "Application Deployed to EKS Successfully!!"
            }
        }
