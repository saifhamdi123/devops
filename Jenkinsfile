pipeline {
    agent any
    environment {
        SONAR_HOST_URL = 'http://192.168.60.4:9000/'
        SONAR_AUTH_TOKEN = credentials('sonarqube')
        NVD_API_KEY = credentials('NVD_API_KEY')
        
    }
    stages {
        stage("Clean Workspace") { 
            steps { 
                cleanWs()
            } 
        }
        stage('Git Checkout') {
            steps {
                echo 'Cloning DevSecOps repository...'
                git branch: 'main',
                    credentialsId: 'gittoken',
                    url: 'https://github.com/saifhamdi123/devops.git'
            }
        }
        
        stage('Secret Scan - Gitleaks') {
            steps {
                echo 'Running Gitleaks secret scan...'
                sh '''
                    mkdir -p gitleaks-report
                    gitleaks detect --source . --report-format json --report-path gitleaks-report/gitleaks-report.json || true
                '''
                archiveArtifacts artifacts: 'gitleaks-report/gitleaks-report.json', allowEmptyArchive: true
            }
        }
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.jar'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage('SAST - SONARQUBE') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv('sonarqube_scanner') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=projet_devsecops \
                        -Dsonar.projectName=projet-DevSecOps \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.token=$SONAR_AUTH_TOKEN \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }
            stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
                    }
                }
            }
        }
        stage("SCA - Dependency Check") {
            steps {
                sh 'mkdir -p dependency-check-report'
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format HTML
                    --format XML
                    --out dependency-check-report
                    --disableYarnAudit 
                    --disableNodeAudit 
                    --nvdApiKey $NVD_API_KEY
                ''',
                odcInstallation: 'DependencyCheck'
                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
            }
        }
        stage('Build Docker Image') {
        steps {
            echo 'Building Docker image for DevSecOps...'
            script {
                env.IMAGE_NAME = 'devops'
                env.IMAGE_TAG = "${IMAGE_NAME}:${BUILD_NUMBER}"
            // Cleanup old images (if exist)
                sh 'docker rmi -f $(docker images devops -q) || true'
                sh "docker image prune -f || true"
                sh "docker build -t ${IMAGE_NAME}:latest ."
                sh "docker tag ${IMAGE_NAME}:latest ${IMAGE_TAG}"
                }
            }
        }
        stage("Trivy Scan Image") {
            steps {
                script {
                    echo 'Running Trivy scan on ${env.IMAGE_TAG}'
                    sh 'mkdir -p trivy-report'
                    sh """
                    trivy image --timeout 30m -f json -o trivy-report/trivy-report.json ${env.IMAGE_TAG}
                    trivy image --timeout 30m -f table -o trivy-report/trivy-report.txt ${env.IMAGE_TAG}
                    """
                    archiveArtifacts artifacts: 'trivy-report/*', allowEmptyArchive: true
                }
            }
        }
        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f devops || true"
                    sh "docker run -d --name devops -p 8084:8080 ${env.IMAGE_TAG}"
                }
            }
        }
        
        
        
        
        
    }
}
