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

        stage('Checkout') {
    steps {
        container('maven') {
            checkout([
                $class: 'GitSCM',
                branches: [[name: '*/main']],
                userRemoteConfigs: [[
                    url: 'https://github.com/fasil7170/netflix-clone.git',
                    credentialsId: 'github-cred'
                ]]
            ])
        }
    }
}

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
                            sh '''
                            mkdir -p ~/.m2

                            cat > ~/.m2/settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>$NEXUS_USER</username>
      <password>$NEXUS_PASS</password>
    </server>
  </servers>
</settings>
EOF

                            mvn versions:set -DnewVersion=1.0.$BUILD_NUMBER
                            mvn deploy -DskipTests
                            '''
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh '''
                    sleep 10
                    docker info

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

        // ✅ FIXED
        stage('Update K8s Manifest') {
            steps {
                container('maven') {
                    sh '''
                    sed -i "s|image:.*|image: $DOCKER_IMAGE:$TAG|" k8s/deployment.yaml
                    '''
                }
            }
        }

        // ✅ NOW PROPERLY SEPARATE STAGE
        stage('Commit & Push Changes') {
    steps {
        container('maven') {
            withCredentials([usernamePassword(
                credentialsId: 'git-cred',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_PASS'
            )]) {
                sh '''
                echo "Workspace check"
                pwd
                ls -la

                git config user.email "rkftrip@gmail.com"
                git config user.name "fasil7170"

                git add k8s/deployment.yaml
                git commit -m "Update image tag" || echo "No changes"

                git push https://${GIT_USER}:${GIT_PASS}@github.com/fasil7170/netflix-clone.git HEAD:main
                '''
            }
        }
    }
}
}
