pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/rrana2208/java-demo.git', description: 'Git Repository URL')
        string(name: 'GITHUB_CREDENTIALS_ID', defaultValue: 'github-java-demo-project-pat', description: 'GitHub PAT Credentials ID')
        string(name: 'ECR_REGISTRY_ID', defaultValue: '521092179643', description: 'AWS ECR Registry ID')
        string(name: 'ECR_REPOSITORY_NAME', defaultValue: 'rrana', description: 'ECR Repository Name')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS Region')
    }

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}" // ✅ moved IMAGE_NAME to be set dynamically in script
    }

    stages {
        stage('Build JAR') {
            steps {
                dir('java-project-code/complete') {
                    sh './mvnw clean package'
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('java-project-code/complete') {
                    sh './mvnw test'
                }
            }
        }

        stage('Login to ECR & Build Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN') // optional
                ]) {
                    script {
                        // ✅ Dynamically set ECR repo based on branch
                        def repoName = 'java-test'
                        if (env.BRANCH_NAME == 'main') {
                            repoName = 'java-prod'
                        }
                        env.IMAGE_NAME = "${params.ECR_REGISTRY_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com/${repoName}"

                        def ecrLoginScript = """
                            set -e
                            echo "🔐 Logging in to ECR..."
                            export AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}
                            ${env.AWS_SESSION_TOKEN ? "export AWS_SESSION_TOKEN=${env.AWS_SESSION_TOKEN}" : ""}
                            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${env.IMAGE_NAME}
                        """
                        sh ecrLoginScript

                        sh """
                            echo "🐳 Building Docker image..."
                            docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} -t ${env.IMAGE_NAME}:latest java-project-code/complete
                        """
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    docker push ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                    docker push ${env.IMAGE_NAME}:latest
                """
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                script {
                    sh """
                        sed -i 's|image: .*|image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}|' k8-deployemnt-java-project/*/java.yaml
                    """
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                dir('java-project-code/complete') {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-credential-id', variable: 'KUBECONFIG'),
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN') // optional
                ]) {
                    script {
                        def namespace = 'java-test' // default namespace
                        if (env.BRANCH_NAME == 'main') {
                            namespace = 'java-prod'
                            manifestFolder = 'k8-deployemnt-java-project/prod'
                        } else if (env.BRANCH_NAME == 'dev') {
                            namespace = 'java-test'
                            manifestFolder = 'k8-deployemnt-java-project/dev'
                        } else {
                            echo "Branch '${env.BRANCH_NAME}' detected - deploying to namespace '${namespace}'"
                        }

                        def updateSecretAndDeploy = """
                            set -e
                            echo "🔐 Refreshing ECR imagePullSecret in namespace '${namespace}'..."
                            export AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}
                            ${env.AWS_SESSION_TOKEN ? "export AWS_SESSION_TOKEN=${env.AWS_SESSION_TOKEN}" : ""}
                            SERVER=${params.ECR_REGISTRY_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com
                            PASSWORD=\$(aws ecr get-login-password --region ${params.AWS_REGION})
                            kubectl create secret docker-registry ecr-creds \\
                              --docker-server=\$SERVER \\
                              --docker-username=AWS \\
                              --docker-password="\$PASSWORD" \\
                              --docker-email=you@example.com \\
                              -n ${namespace} --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=\$KUBECONFIG

                            echo "🚀 Deploying to Kubernetes namespace '${namespace}' using mainfest folder"
                            kubectl apply -n ${namespace} -f ${manifestFolder} --kubeconfig=\$KUBECONFIG
                        """
                        sh updateSecretAndDeploy
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                emailext(
                    to: 'always@foo.com',
                    recipientProviders: [previous()],
                    subject: "Build ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p><strong>Build Status:</strong> ${buildStatus}</p>
                        <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    mimeType: 'text/html'
                )
            }
        }
        success {
            echo "✅ Pipeline completed and deployed: ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed."
        }
    }
}

