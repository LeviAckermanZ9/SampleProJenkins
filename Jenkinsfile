pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  parameters {
    string(
      name: 'DOCKERHUB_REPO',
      defaultValue: 'your-dockerhub-username/sampleprojenkins',
      description: 'Docker Hub image name, in username/repository format.'
    )
    string(
      name: 'EC2_HOST',
      defaultValue: '',
      description: 'EC2 public DNS name or IP address; required only when deployment is enabled.'
    )
    booleanParam(
      name: 'DEPLOY_TO_EC2',
      defaultValue: false,
      description: 'Deploy the published image to the EC2_HOST configured in Jenkins.'
    )
  }

  environment {
    DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
    EC2_SSH_CREDENTIALS = 'ec2-ssh-key'
    EC2_CONTAINER_NAME = 'sampleprojenkins'
  }

  stages {
    stage('Validate configuration') {
      steps {
        script {
          env.EC2_HOST = params.EC2_HOST?.trim()
          if (!(params.DOCKERHUB_REPO ==~ /^[a-z0-9]+(?:[._-][a-z0-9]+)*\/[a-z0-9]+(?:[._-][a-z0-9]+)*$/) || params.DOCKERHUB_REPO.startsWith('your-dockerhub-username/')) {
            error('Set DOCKERHUB_REPO to a valid Docker Hub username/repository, for example myname/sampleprojenkins.')
          }
          if (params.DEPLOY_TO_EC2 && !(params.EC2_HOST ==~ /^[A-Za-z0-9][A-Za-z0-9.-]*$/)) {
            error('Set EC2_HOST to the instance public DNS name or IP address before enabling EC2 deployment.')
          }
        }
      }
    }

    stage('Build image') {
      steps {
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
          env.IMAGE_NAME = "${params.DOCKERHUB_REPO}"
        }
        sh 'docker build --pull -t "$IMAGE_NAME:$IMAGE_TAG" -t "$IMAGE_NAME:latest" .'
      }
    }

    stage('Publish to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: env.DOCKERHUB_CREDENTIALS,
          usernameVariable: 'DOCKERHUB_USER',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            set +x
            trap 'docker logout >/dev/null 2>&1 || true' EXIT
            printf '%s' "$DOCKERHUB_TOKEN" | docker login --username "$DOCKERHUB_USER" --password-stdin
            docker push "$IMAGE_NAME:$IMAGE_TAG"
            docker push "$IMAGE_NAME:latest"
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      when {
        expression { params.DEPLOY_TO_EC2 }
      }
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: env.EC2_SSH_CREDENTIALS,
          keyFileVariable: 'EC2_SSH_KEY',
          usernameVariable: 'EC2_SSH_USER'
        )]) {
          sh '''
            set -eu
            test -f "$HOME/.ssh/known_hosts" || {
              echo 'Add the EC2 host key to the Jenkins agent known_hosts file before deploying.' >&2
              exit 1
            }
            ssh -i "$EC2_SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes "$EC2_SSH_USER@$EC2_HOST" \
              "docker pull '$IMAGE_NAME:$IMAGE_TAG' && (docker rm -f '$EC2_CONTAINER_NAME' >/dev/null 2>&1 || true) && docker run -d --restart unless-stopped --name '$EC2_CONTAINER_NAME' -p 80:80 '$IMAGE_NAME:$IMAGE_TAG'"
          '''
        }
      }
    }
  }

  post {
    always {
      script {
        if (env.IMAGE_NAME && env.IMAGE_TAG) {
          sh 'docker image rm "$IMAGE_NAME:$IMAGE_TAG" "$IMAGE_NAME:latest" 2>/dev/null || true'
        }
      }
    }
  }
}
