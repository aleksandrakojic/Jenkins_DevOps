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
                    def dockerTag = ""
                    if (env.BRANCH_NAME == 'dev') {
                        dockerTag = "dev"
                    } else if (env.BRANCH_NAME == 'qa') {
                        dockerTag = "qa"
                    } else if (env.BRANCH_NAME == 'staging') {
                        dockerTag = "staging"
                    } else if (env.BRANCH_NAME == 'master') {
                        dockerTag = "latest"
                    }
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKERHUB_CREDENTIALS}") {
                        echo 'Starting in Docker.'
                        // Build and push movie-service image
                        def movieService = docker.build("${MOVIE_IMAGE}:${dockerTag}", "./movie-service")
                        sleep 6
                        movieService.push()
                        echo 'Pushed movie service to Docker.'
                        sleep 6

                        // Build and push cast-service image
                        def castService = docker.build("${CAST_IMAGE}:${dockerTag}", "./cast-service")
                        sleep 6
                        castService.push()
                        echo 'Pushed cast service to Docker.'
                        sleep 6
                    }
                }
            }
        }
        stage('Deploy to Environments') {
            steps {
                script {
                    def namespace = ""
                    def valuesFile = ""
                    
                    switch (env.BRANCH_NAME) {
                        case "dev":
                            namespace = "dev"
                            valuesFile = "values-dev.yaml"
                            break
                        case "qa":
                            namespace = "qa"
                            valuesFile = "values-qa.yaml"
                            break
                        case "staging":
                            namespace = "staging"
                            valuesFile = "values-staging.yaml"
                            break
                    }

                    if (namespace && valuesFile) {
                        // Deploy cast service
                        sh "helm upgrade --install jenkins-${namespace} ./jenkins-charts --namespace ${namespace} --values ./jenkins-charts/${valuesFile} --set image.tag=${env.BUILD_NUMBER}"
                    }
                }
            }
        }
        stage('Deployment to Production') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh "helm upgrade --install jenkins-prod ./jenkins-charts --namespace prod --values ./jenkins-charts/values-prod.yaml --set image.tag=${env.BUILD_NUMBER}"
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'The pipeline has finished.'
            cleanWs() 
        }
    }
}
