
 pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "fazil2664/user-service"
        NEXUS_URL = "http://192.168.0.100:8081"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/netflix-clone.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                sh 'mvn deploy -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE:latest user-service/"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                    sh "docker push $DOCKER_IMAGE:latest"
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                sh """
                sed -i 's|image:.*|image: $DOCKER_IMAGE:latest|' k8s/deployment.yaml
                """
            }
        }

        stage('Commit & Push Changes') {
            steps {
                sh """
                git config user.email "jenkins@local"
                git config user.name "jenkins"
                git add .
                git commit -m "Updated image"
                git push origin main
                """
            }
        }
    }
}
