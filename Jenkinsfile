pipeline {
    agent any
    environment {
        // Credentials
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials') 
        // Image repositories
        CAST_IMAGE = "underdogdevops/cast-service"
        MOVIE_IMAGE = "underdogdevops/movie-service"
        // Docker Hub Username
        DOCKER_USERNAME = 'underdogdevops' 
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

                    // Docker login
                    sh "echo ${env.DOCKER_CREDENTIALS_PSW} | docker login -u ${env.DOCKER_CREDENTIALS_USR} --password-stdin"

                    // Parallel building and pushing of images
                    parallel(
                        buildAndPushMovieService: {
                            sh "docker build -t ${env.MOVIE_IMAGE}:${dockerTag} -f ./movie-service/Dockerfile ./movie-service"
                            sh "docker push ${env.MOVIE_IMAGE}:${dockerTag}"
                        },
                        buildAndPushCastService: {
                            sh "docker build -t ${env.CAST_IMAGE}:${dockerTag} -f ./cast-service/Dockerfile ./cast-service"
                            sh "docker push ${env.CAST_IMAGE}:${dockerTag}"
                        }
                    )

                    // Docker logout
                    sh "docker logout"
                }
            }
        }
        stage('Install Helm') {            
            steps {
                script {
                    // Install Helm
                    sh "curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3"
                    sh "chmod 700 get_helm.sh"
                    sh "./get_helm.sh"
                    sh "echo helm version --client"
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
                    // Helm upgrade/install
                    sh "helm upgrade --install ${helmReleaseName} ./jenkins-charts --namespace ${env.BRANCH_NAME} --values ./jenkins-charts/${valuesFile} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
        stage('Deployment to Production') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy to production?', ok: 'Yes'
                }
                script {
                    // Helm upgrade/install for production
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
