pipeline {
  agent any
  environment {
    APP_NAME = 'sample-node-app'
    DOCKER_URL = '772825290081.dkr.ecr.ap-southeast-1.amazonaws.com'
    DOCKER_CREDENTIALS = 'ecr:ap-southeast-1:jenkins-user-new'
  }
  stages {
    stage('Git clone') {
      steps {
        git url: 'https://github.com/tanmay-pnaik/sample-node-app.git'
      }
    }
    stage('Docker build') {
        steps {
            script {
              docker.build(APP_NAME)
            }
        }
    }
    stage('Docker push') {
      steps {
          script {
            docker.withRegistry('https://${DOCKER_URL}', env.DOCKER_CREDENTIALS) {
                docker.image(APP_NAME).push(BUILD_NUMBER)
            }
          }
      }
    }
    stage ('Deploy') {
      steps {
        sh '''
        aws eks --region ap-southeast-1 update-kubeconfig --name frute-backend
        kubectl config set-context --current --namespace=jenkins
        kubectl apply -f /home/ec2-user/sample-node-app-deployment.yaml
        '''
      }
    }
    stage('Cleanup') {
      steps{
        sh "docker rmi ${DOCKER_URL}/${APP_NAME}:${BUILD_NUMBER}"
      }
    }
  }
}
