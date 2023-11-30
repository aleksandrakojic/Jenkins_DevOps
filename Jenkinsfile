pipeline {
    agent any
    environment {
        // Credentials
        DOCKER_CREDENTIALS = credentials('DOCKER_HUB_PASS') 
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
                    sh "echo ${DOCKER_CREDENTIALS} | docker login -u ${DOCKER_USERNAME} --password-stdin"

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
        stage('Deploy to Environments') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'qa'
                    branch 'staging'
                }
            }
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            steps {
                script {                    
                    def valuesFile = "values-${env.BRANCH_NAME}.yaml"
                    // Helm upgrade/install
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config                    
                    '''
                    sh "helm upgrade --install jenkins-devops ./jenkins-charts --namespace ${env.BRANCH_NAME} --values ./jenkins-charts/${valuesFile} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
        stage('Deployment to Production') {
            when {
                branch 'master'
            }
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy to production?', ok: 'Yes'
                }
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    '''
                    sh "helm upgrade --install jenkins-devops ./jenkins-charts --namespace prod --values ./jenkins-charts/values-prod.yaml --set image.tag=${env.BUILD_NUMBER}"
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
