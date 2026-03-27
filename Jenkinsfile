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
    command: ['cat']
    tty: true

  - name: docker
    image: docker:24.0.5-dind
    securityContext:
      privileged: true
    command:
    - dockerd-entrypoint.sh
    tty: true
"""
        }
    }

    environment {
        DOCKER_IMAGE = "fazil2664/user-service"
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Build') {
    steps {
        container('maven') {
            dir('user-service') {
                sh '''
                mvn clean package -DskipTests
                ls -la target
                '''
            }
        }
    }
}

        stage('Test') {
            steps {
                container('maven') {
                    dir('user-service') {
                        sh 'mvn test'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    dir('user-service') {
                        withSonarQubeEnv('sonar-server') {
                            sh 'mvn sonar:sonar'
                        }
                    }
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                container('maven') {
                    withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        dir('user-service') {
                            sh """
                            mkdir -p ~/.m2

                            cat > ~/.m2/settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
EOF

                            echo "Setting dynamic version..."
                            mvn versions:set -DnewVersion=1.0.$BUILD_NUMBER

                            mvn deploy -DskipTests
                            """
                        }
                    }
                }
            }
        }

       stage('Build Docker Image') {
    steps {
        container('docker') {
            sh '''
            echo "Waiting for Docker daemon..."
            sleep 10
            docker info

            echo "Checking JAR file..."
            ls -la user-service/target

            cd user-service
            docker build -t $DOCKER_IMAGE:$TAG .
            '''
        }
    }
}

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $DOCKER_IMAGE:$TAG
                        '''
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                sh """
                sed -i 's|image:.*|image: $DOCKER_IMAGE:$TAG|' k8s/deployment.yaml
                """
            }
        }

       stage('Commit & Push Changes') {
    steps {
        container('maven') {

            // 🔥 THIS IS REQUIRED
            checkout scm

            withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {

                sh '''
                pwd
                ls -la

                git config user.email "jenkins@local"
                git config user.name "jenkins"

                git add .
                git commit -m "Updated image to '$TAG'" || echo "No changes"
                git push origin main
                '''
            }
        }
    }
}
}
