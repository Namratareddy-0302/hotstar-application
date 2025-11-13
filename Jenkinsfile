pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_DEFAULT_REGION = 'us-east-1'
        CLUSTER_NAME = 'eks-cluster4'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Hotstar \
                        -Dsonar.projectKey=Hotstar
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey 80C712B1-20C0-F011-8362-129478FCB64D', odcInstallation: 'DC'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh "trivy fs . > trivyfs.txt || true"
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t hotstar ."
                        sh "docker tag hotstar skr6528/hotstar:latest"
                        sh "docker push skr6528/hotstar:latest"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh "trivy image skr6528/hotstar:latest > trivyimage.txt || true"
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                    docker rm -f hotstar || true
                    docker run -d --name hotstar -p 80:3000 skr6528/hotstar:latest
                '''
            }
        }

        stage('Configure Kubeconfig') {
            steps {
                sh '''
                    echo "Configuring kubeconfig for EKS cluster..."
                    aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $CLUSTER_NAME
                    echo "Verifying kubectl connectivity..."
                    kubectl cluster-info
                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    echo "Applying manifests..."
                    kubectl apply -f K8S/manifest.yml
                '''
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins HOTSTAR CICD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
