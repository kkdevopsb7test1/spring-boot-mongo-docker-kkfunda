pipeline {
    agent any

    tools {
        maven 'maven3.9.9'
    }

    environment {
        SNYK_TOKEN = credentials('SNYK_TOKEN') // Snyk API token
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning Repository..."
                git branch: 'feature',
                    url: 'https://github.com/Pvreddy1/spring-boot-mongo-docker-kkfunda.git'
            }
        }

        stage('Maven Build') {
            steps {
                echo "Building the Maven Project..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    echo "Running SonarQube analysis..."
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=spring-boot-mongo \
                        -Dsonar.projectName="Spring Boot Mongo Project"
                    '''
                }
            }
        }

        stage('Snyk Repo Scan') {
            steps {
                echo "Running Snyk scan for dependency vulnerabilities..."
                sh '''
                    echo "Authenticating with Snyk..."
                    snyk auth $SNYK_TOKEN
                    echo "Running Snyk Test..."
                    snyk test --all-projects || true
                    snyk monitor --all-projects || true
                '''
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker build -t pvreddy1/mongospring:latest ."
                    }
                }
            }
        }


        stage('Trivy Image Scan') {
            steps {
                echo "Scanning Docker image for vulnerabilities using Trivy..."
                sh '''
                    trivy image --exit-code 0 --severity HIGH,CRITICAL pvreddy1/mongospring:latest
                    trivy image --exit-code 1 --severity CRITICAL pvreddy1/mongospring:latest || true
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker image to DockerHub..."
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push pvreddy1/mongospring:latest"
                    }
                }
            }
        }

        stage('Setup KubeConfig') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-cred']]) {
                    sh '''
                        echo "Setting up kubeconfig for EKS cluster..."

                        # Export AWS credentials explicitly
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=ap-south-1

                        echo "Verifying AWS Identity..."
                        aws sts get-caller-identity

                        echo "Updating kubeconfig for EKS cluster..."
                        aws eks update-kubeconfig --region ap-south-1 --name my-cluster

                        echo "Kubeconfig setup complete."
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying application to EKS..."
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-cred']]) {
                    sh '''
                        kubectl apply -f springappmongo.yaml --validate=false
                    '''
                }
            }
        }

        stage('Verify Pods and Services') {
            steps {
                echo "Verifying Kubernetes resources..."
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-cred']]) {
                    sh '''
                        kubectl get pods -o wide
                        kubectl get svc -o wide
                    '''
                }
            }
        }

        stage('Trivy Cluster Scan') {
            steps {
                echo "Scanning Kubernetes cluster for vulnerabilities..."
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-cred']]) {  
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=ap-south-1

                        echo "Running Trivy Kubernetes cluster scan..."
                        trivy k8s --report summary cluster || true
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Please check the logs for details."
        }
    }
}
