node {
  stage 'Checkout'
  git 'https://github.com/tanmay-pnaik/sample-node-app.git'

  stage 'Docker build'
  docker.build('sample-node-app')

  stage 'Docker push'
  docker.withRegistry('https://772825290081.dkr.ecr.ap-southeast-1.amazonaws.com', 'ecr:ap-southeast-1:jenkins-user-new') {
    docker.image('sample-node-app').push('${BUILD_NUMBER}')
  }
}