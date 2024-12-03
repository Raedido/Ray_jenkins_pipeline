pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', 
                    credentialsId: 'git-cred', 
                    url: 'https://github.com/Raedido/Ray_jenkins_pipeline.git'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn -X compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Trivy fs scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Blogging-app \
                        -Dsonar.projectKey=Blogging-app \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Publish Artifacts') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'artifact-repo-cred', usernameVariable: 'ARTIFACT_USERNAME', passwordVariable: 'ARTIFACT_PASSWORD')
                ]) {
                    withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'maven3') {
                        sh '''
                            mvn deploy \
                            -Drepository.username=$ARTIFACT_USERNAME \
                            -Drepository.password=$ARTIFACT_PASSWORD
                        '''
                    }
                }
            }
        }
        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        // Docker build command to build the image
                        sh 'docker build -t raedido/bloggingapp:latest . --no-cache'
                    }
                }
            }
        }
        stage('Trivy image scan') {
            steps {
                sh 'trivy image --format table -o image.html divyasatpute/bloggingapp:latest'
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker push raedido/bloggingapp:latest'
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8-cred']) {
                        sh '''
                        kubectl apply -f deployment-service.yml
                        DEPLOYMENT_NAME=$(kubectl get deployment -n webapps --selector=app=my-app -o jsonpath="{.items[0].metadata.name}")
                        kubectl rollout status deployment/$DEPLOYMENT_NAME -n webapps --timeout=60s
                        '''
                    }
                }
            }
        }
 
        stage('Verify Kubernetes Deployment') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8-cred']) {
                        sh '''
                        kubectl get pods -n webapps
                        kubectl get svc -n webapps
                        '''
                    }
                }
            }
        }
    }
 
    post {
        always {
            echo 'Pipeline completed.'
            cleanWs() // Clean up the workspace after the pipeline
        }
 
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
 
        success {
            echo 'Pipeline completed successfully.'
        }
    }
}
