pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

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
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
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
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey 4605998F-BEC0-F011-8365-0EBF96DE670D', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    sh "docker build -t hotstar ."
                    sh "docker tag hotstar namratareddy/hotstar:latest"
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image namratareddy/hotstar:latest > trivyimage.txt"
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: '', toolName: 'docker') {
                        sh "docker push namratareddy/hotstar:latest"
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    // Remove dangling Docker images
                    sh "docker system prune -f"

                    // Remove local scan files to save space
                    sh "rm -f trivyfs.txt trivyimage.txt"
                }
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'GitHub User'

                // Only attach files if they exist
                def attachments = []
                if (fileExists('trivyfs.txt')) attachments.add('trivyfs.txt')
                if (fileExists('trivyimage.txt')) attachments.add('trivyimage.txt')

                emailext (
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins HOTSTAR CICD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'reddynamrata99@gmail.com',
                    from: 'reddynamrata99@gmail.com',
                    replyTo: 'reddynamrata99@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: attachments.join(',')
                )
            }
        }
    }
}
