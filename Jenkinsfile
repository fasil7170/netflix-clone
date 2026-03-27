pipeline {
    agent {
        kubernetes {
            label 'devops-agent'
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9.9-eclipse-temurin-17
    command:
    - cat
    tty: true

  - name: docker
    image: docker:24.0.5
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock

  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        DOCKER_IMAGE = "fazil2664/user-service"
        TAG = "${BUILD_NUMBER}"
        GIT_REPO = "https://github.com/fasil7170/netflix-clone.git"
    }

    stages {

        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('sonar-server') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                container('maven') {
                    sh 'mvn deploy -DskipTests'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh "docker build -t $DOCKER_IMAGE:$TAG user-service/"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $DOCKER_IMAGE:$TAG
                        """
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                container('maven') {
                    sh """
                    sed -i 's|image:.*|image: $DOCKER_IMAGE:$TAG|' k8s/deployment.yaml
                    """
                }
            }
        }

        stage('Commit & Push Changes') {
            steps {
                container('maven') {
                    withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                        git config user.email "jenkins@local"
                        git config user.name "jenkins"

                        git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/fasil7170/netflix-clone.git

                        git add .
                        git commit -m "Updated image to $TAG" || echo "No changes to commit"
                        git push origin main
                        """
                    }
                }
            }
        }
    }
}
