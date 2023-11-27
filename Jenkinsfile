pipeline {
  agent any
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

    stage('install helm') {
      steps {
        sh 'wget https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz'
        sh 'ls -a'
        sh 'tar -xvzf helm-v3.6.1-linux-amd64.tar.gz'
        sh 'sudo cp linux-amd64/helm /usr/bin'
        sh 'helm version'
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
        timeout(time: 5, unit: 'MINUTES') {
          input(message: 'Do you want to deploy to production?', ok: 'Yes')
        }

        script {
          sh "helm upgrade --install jenkins-prod ./jenkins-charts --namespace prod --values ./jenkins-charts/values-prod.yaml --set image.tag=${env.BUILD_NUMBER}"
        }

      }
    }

  }
  environment {
    CAST_IMAGE = 'underdogdevops/cast-service'
    MOVIE_IMAGE = 'underdogdevops/movie-service'
    DOCKER_USERNAME = 'underdogdevops'
  }
  post {
    always {
      echo 'The pipeline has finished.'
      cleanWs()
    }

    failure {
      echo 'The pipeline failed!'
    }

  }
}