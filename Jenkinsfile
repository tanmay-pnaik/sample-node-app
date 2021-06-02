def COLOR_MAP = [
    'SUCCESS': '32CD32',
    'FAILURE': 'FF0000',
]

pipeline {
  agent any
  environment {
    APP_NAME = 'sample-node-app'
    DOCKER_REPO_NAME = 'rakbank-customer-service-jenkins-qa'
    DEPLOYMENT_FILE_NAME = 'sample-node-app-deployment.yaml'
    DOCKER_REPO_URL = '772825290081.dkr.ecr.ap-southeast-1.amazonaws.com'
    DOCKER_REPO_CREDENTIALS = 'ecr:ap-southeast-1:jenkins-user'
    CLUSTER_NAMESPACE = 'jenkins'
    GIT_BRANCH_NAME = "${GIT_BRANCH}"
  }

  options {
    // Keep only the last 20 builds
    buildDiscarder(logRotator(numToKeepStr: '20'))

    // Timeout of 1 hour for entire build
    timeout(time: 1, unit: 'HOURS')

    disableConcurrentBuilds()

    office365ConnectorWebhooks([[
        startNotification: true,
        url: env.MS_TEAMS_WEBHOOK_URL,
        notifySuccess: true,
        notifyAborted: true,
        notifyNotBuilt: true,
        notifyUnstable: true,
        notifyFailure: true,
        notifyBackToNormal: true,
        notifyRepeatedFailure: true,
        timeout: 30000
      ]]
    )
  }

  stages {
    stage('Initialization') {
      steps {
        script {
          // Fetch current tag from ECR and compute new tag name
          env.DOCKER_IMAGE_TAG = sh (
              script: '''
                # Fetching current tag from ECR
                CURRENT_TAG=$(aws ecr describe-images \
                --repository-name ${DOCKER_REPO_NAME} \
                --output text \
                --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]' \
                | tr '\t' '\n' \
                | tail -1)

                # Fetching correct current tag from ECR if "latest"
                if [ "$CURRENT_TAG" = "latest" ]
                    then
                        CURRENT_TAG=$(aws ecr describe-images \
                        --repository-name ${DOCKER_REPO_NAME} \
                        --output text \
                        --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]' \
                        | tr '\t' '\n' \
                        | tail -n 2  \
                        | head -n 1)
                fi

                # Extracting only the last point release number
                ID=${CURRENT_TAG##*.}

                # Incrementing by 1
                NEW_ID=$((ID+1))

                # Computing NEW_TAG by appending NEW_ID to "rakbank-v1.2"
                NEW_TAG=${CURRENT_TAG%.*}.$NEW_ID
                echo $NEW_TAG
              ''',
              returnStdout: true
          ).trim()
          echo "New tag for repo ${DOCKER_REPO_NAME} is: ${DOCKER_IMAGE_TAG}"

          // Fetching author, commit ID and commit message of latest commit
          gitAuthor = sh(
            returnStdout: true,
            script: "git --no-pager show -s --format='%an' ${GIT_COMMIT}"
          ).trim()
          gitCommitId = sh(returnStdout: true, script: "git rev-parse --short ${GIT_COMMIT}").trim()
          gitCommitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
          echo 'Last git commit details:'
          echo 'Author: ' + gitAuthor
          echo 'Commit ID: ' + gitCommitId
          echo 'Commit mesage: ' + gitCommitMessage
        }
      }
    }

    stage('Git') {
      steps {
        echo "App name is ${APP_NAME}"
        git branch: 'dev',
            url: 'https://github.com/tanmay-pnaik/sample-node-app.git'
      }
    }

    stage('Docker build') {
      steps {
        script {
          // --network=host lets Docker know that it can use host network
          sh "docker build --network=host -t ${DOCKER_REPO_NAME} ."
        }
      }
    }
    stage('Docker push') {
      steps {
        script {
          echo "Tagging image of ${DOCKER_REPO_NAME} as ${env.DOCKER_IMAGE_TAG}"
          echo "Workspace is ${WORKSPACE}"
          // Pushing to AWS ECR Docker registry
          docker.withRegistry('https://${DOCKER_REPO_URL}', env.DOCKER_REPO_CREDENTIALS) {
            // Choosing the image and the registry
            def dockerImage = docker.image(env.DOCKER_REPO_NAME)

            // Tagging with custom tag number (sent as build parameter)
            dockerImage.push(env.DOCKER_IMAGE_TAG)

            // Tagging as 'latest'
            dockerImage.push('latest')
          }
        }
      }
    }
    stage ('Deploy') {
      steps {
        // Updating KUBECONFIG, setting namespace and applying deployment YAML file
        sh '''
          aws eks --region ap-southeast-1 update-kubeconfig --name frute-backend
          kubectl config set-context --current --namespace=${CLUSTER_NAMESPACE}
          kubectl apply -f ${WORKSPACE}/${DEPLOYMENT_FILE_NAME}
          kubectl patch deployment sample-node-app -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
        '''
      }
    }
  }
  post {
    cleanup {
      // Deleting locally stored Docker images
      sh '''
        docker rmi ${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:${DOCKER_IMAGE_TAG}
        docker rmi ${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:latest
      '''
    }
    always {
      // Sending notification to MS Teams
      office365ConnectorSend (
        webhookUrl: env.MS_TEAMS_WEBHOOK_URL,
        color: COLOR_MAP[currentBuild.currentResult],
        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}<br /><br />" +
                "SERVICE: ${APP_NAME}<br /><br />" +
                "BRANCH: ${GIT_BRANCH_NAME}<br /><br />" +
                "TAG NAME: ${DOCKER_IMAGE_TAG}<br /><br />" +
                "NAMESPACE: ${CLUSTER_NAMESPACE}<br /><br />" +
                "TIME TAKEN: ${currentBuild.durationString.replace(' and counting', '')}<br /><br />" +
                'LAST COMMIT DETAILS: <br />' +
                '- Author: ' + gitAuthor + '<br />' +
                '- Commit ID: ' + gitCommitId + '<br />' +
                '- Commit mesage: ' + gitCommitMessage + '<br />'
      )
    }
  }
}
