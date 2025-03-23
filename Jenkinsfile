pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Devops2022jk/netflix.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                script {
                    def exitCode = sh(script: "trivy fs . > trivyfs.txt", returnStatus: true)
                    if (exitCode != 0) {
                        echo "Trivy scan failed, but proceeding automatically..."
                    }
                }
            }
        }
        stage("Cleanup Old Docker Containers & Images") {
            steps {
                script {
                    sh '''
                    if [ "$(docker ps -q -f name=netflix)" ]; then
                        echo "Stopping and removing existing Netflix container..."
                        docker stop netflix || true && docker rm netflix || true
                    else
                        echo "No existing Netflix container found."
                    fi
                    '''
                    sh '''
                    if [ "$(docker images -q netflix)" ]; then
                        echo "Removing old Netflix image..."
                        docker rmi -f netflix || true
                    else
                        echo "No old Netflix image found."
                    fi
                    '''
                }
            }
        }
        stage("Docker Build Image") {
            steps {
                withCredentials([string(credentialsId: 'API_KEY', variable: 'netflix_key')]) {
                    sh "docker build --build-arg TMDB_V3_API_KEY=$netflix_key -t netflix ."
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps {
                script {
                    def exitCode = sh(script: "trivy image netflix > trivyimage.txt", returnStatus: true)
                    if (exitCode != 0) {
                        echo "Trivy image scan failed, but proceeding automatically..."
                    }
                }
            }
        }
        stage("Docker Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker_cred', toolName: 'docker') {   
                        sh "docker tag netflix devopsjk2023/netflix:latest"
                        sh "docker push devopsjk2023/netflix:latest"
                    }
                }
            }
        }
        stage("Deploy Container") {
            steps {
                sh "docker container run -it -d -p 96:80 --name netflix devopsjk2023/netflix:latest"
            }
        }
    }
}
