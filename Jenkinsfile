pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPO', defaultValue: 'git@github.com:pearceja/text-generation-webui.git', description: 'Git repository URL')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')
        string(name: 'DOCKER_REGISTRY', defaultValue: 'gcr.io', description: 'Docker registry URL')
        string(name: 'PROJECT_ID', defaultValue: 'gwaan-ai', description: 'Google Cloud Project ID')
        string(name: 'REGION', defaultValue: 'europe-west2', description: 'Google Cloud Region')
        string(name: 'REPO_NAME', defaultValue: 'text-generation-webui', description: 'Repository name for Docker image')
        string(name: 'CREDENTIALS_ID', defaultValue: 'gcp-service-account-key', description: 'Credentials ID for Google Cloud')
        string(name: 'SSH_CREDENTIALS_ID', defaultValue: 'tickle', description: 'SSH Credentials ID for Git operations')
        booleanParam(name: 'USE_HELM', defaultValue: false, description: 'Use Helm Chart management')
    }

    environment {
        IMAGE_NAME = "${params.REPO_NAME}"
        TAG = "${BUILD_NUMBER}-${env.GIT_COMMIT[0..7]}"
        CHART_NAME = "webui-app" // Adjust this as needed
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        userRemoteConfigs: [[credentialsId: params.SSH_CREDENTIALS_ID, url: params.GIT_REPO]]])
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${params.DOCKER_REGISTRY}/${params.PROJECT_ID}/${IMAGE_NAME}:${TAG} ."
            }
        }

        stage('Authenticate and configure Docker') {
            steps {
                withCredentials([string(credentialsId: params.CREDENTIALS_ID, variable: 'GCP_SA_KEY')]) {
                    writeFile file: 'key.json', text: env.GCP_SA_KEY
                    sh "gcloud auth activate-service-account --key-file=key.json"
                    sh "gcloud --quiet auth configure-docker ${params.REGION}-docker.pkg.dev"
                }
            }
        }

        stage('Push to Artifact Registry') {
            steps {
                script {
                    def dockerImage = "${params.DOCKER_REGISTRY}/${params.PROJECT_ID}/${IMAGE_NAME}:${TAG}"
                    try {
                        sh 'docker login -u _json_key --password-stdin https://${params.DOCKER_REGISTRY} < key.json'
                        sh "docker push ${dockerImage}"
                    } catch (Exception e) {
                        error("Failed to push Docker image: ${e.message}")
                    }
                }
            }
        }

        stage('Chart Promotion') {
            when {
                expression { params.USE_HELM == true }
                branch 'staging'
            }
            steps {
                echo "Helm chart update steps would go here."
                // Implement your Helm chart update logic here
            }
        }
    }

    post {
        always {
            echo 'Cleaning up and finishing the pipeline.'
            // Implement any necessary post-build actions here
        }
    }
}
