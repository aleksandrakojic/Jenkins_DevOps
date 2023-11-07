pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-credentials')
        CAST_IMAGE = "underdogdevops/cast-service"
        MOVIE_IMAGE = "underdogdevops/movie-service"
    }
    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }
        stage('Build and push images') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'qa'
                    branch 'staging'
                    branch 'master' 
                }
            }
            steps {
                script {
                    def dockerTag = env.BRANCH_NAME == 'master' ? 'latest' : env.BRANCH_NAME
                    // Log in to Docker Hub
                    sh '''
                    docker build -t ${MOVIE_IMAGE}:${dockerTag} ./movie-service/Dockerfile
                    docker build -t ${CAST_IMAGE}:${dockerTag} ./cast-service/Dockerfile
                    sleep 10

                    docker login -u underdogdevops -p $DOCKERHUB_CREDENTIALS
                    sleep 5
                    
                    docker push ${MOVIE_IMAGE}:${dockerTag}
                    docker push ${CAST_IMAGE}:${dockerTag}
                    '''
                }
            }
        }
        stage('Deploy to Environments') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'qa'
                    branch 'staging'
                }
            }
            steps {
                script {
                    def helmReleaseName = "jenkins-${env.BRANCH_NAME}"
                    def valuesFile = "values-${env.BRANCH_NAME}.yaml"
                    // Deploy to respective namespace based on branch name
                    sh "helm upgrade --install ${helmReleaseName} ./jenkins-charts --namespace ${env.BRANCH_NAME} --values ./jenkins-charts/${valuesFile} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
        stage('Deployment to Production') {
            when {
                branch 'master' // or 'main' depending on your main branch name
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy to production?', ok: 'Yes'
                }
                script {
                    // Deploy to production
                    sh "helm upgrade --install jenkins-prod ./jenkins-charts --namespace prod --values ./jenkins-charts/values-prod.yaml --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
    }
    post {
        always {
            echo 'The pipeline has finished.'
            cleanWs() // Cleans the workspace
        }
        failure {
            echo 'The pipeline failed!'
        }
    }
}
