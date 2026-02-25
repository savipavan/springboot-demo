pre-requisites:
Docker desktop to be installed

Create 
Dockerfile
```dockerfile
FROM jenkins/jenkins:lts

USER root

# Install Docker CLI
RUN apt-get update && \
    apt-get install -y docker.io curl maven && \
    rm -rf /var/lib/apt/lists/*

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl

USER jenkins
```
docker build -t jenkins-docker-k8s .

docker run -d --name jenkins  -u root -p 8080:8080 -p 50000:50000  -v jenkins_home:/var/jenkins_home   -v //var/run/docker.sock:/var/run/docker.sock   -v %USERPROFILE%\.kube:/root/.kube  jenkins-docker-k8s

Access URL with localhost:8080 --> enter jenkins password --> install suggested plugins
``` Jenkins password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Create pipeline 

```pipeline

pipeline {
    agent any

    environment {
        IMAGE_NAME = "demo-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/savipavan/springboot-demo.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                kubectl set image deployment/demo-app \
                demo=${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'kubectl rollout status deployment/demo-app'
                sh 'kubectl get pods'
            }
        }
    }
}
```
Access application using local URL : http://localhost:30007/
