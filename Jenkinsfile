pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "pravinkr11/medicure:latest"
        DOCKER_CREDENTIALS = 'docker-hub-credentials'
        KUBE_CONFIG = credentials('kubeconfig')
        ANSIBLE_HOSTS = '/etc/ansible/hosts'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git 'https://github.com/pravinkr11/medicure.git'
            }
        }

        stage('Build & Test with Maven') {
            steps {
                sh 'mvn clean package'
                sh 'mvn test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker tag $DOCKER_IMAGE pravinkr11/medicure:latest"
                    sh "docker push pravinkr11/medicure:latest"
                }
            }
        }

        stage('Provision Infrastructure with Terraform') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                }
            }
        }

        stage('Configure Kubernetes with Ansible') {
            steps {
                sh "ansible-playbook -i $ANSIBLE_HOSTS deploy.yml"
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl apply -f k8s/dev-deployment.yml'
                }
            }
        }

        stage('Run Automated Tests with Selenium') {
            steps {
                sh "python3 test_selenium.py"
            }
        }

        stage('Deploy to Stage Environment (On Approval)') {
            steps {
                input message: "Deploy to Stage?", ok: "Proceed"
                sh 'kubectl apply -f k8s/stage-deployment.yml'
            }
        }

        stage('Deploy to Prod Environment (On Approval)') {
            steps {
                input message: "Deploy to Prod?", ok: "Proceed"
                sh 'kubectl apply -f k8s/prod-deployment.yml'
            }
        }

        stage('Monitor with Prometheus & Grafana') {
            steps {
                sh "kubectl apply -f monitoring/prometheus.yml"
                sh "kubectl apply -f monitoring/grafana.yml"
            }
        }

        stage('Verify Deployment') {
            steps {
                sh "kubectl get pods -o wide"
                sh "kubectl get services"
            }
        }
    }
}
