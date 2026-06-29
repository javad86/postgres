pipeline {
  agent any

  environment {
    IMAGE_NAME = 'javad86/postgres'
    IMAGE_TAG = "${BUILD_NUMBER}"
    DEPLOY_NAMESPACE = 'default'
    KUBECONFIG_CREDENTIALS = 'rancher-kubeconfig'
    POSTGRES_PASSWORD_CREDENTIAL = 'postgres-password'
    DEPLOYMENT_NAME = 'postgres'
  }

  stages {
    stage('Build Docker Image') {
      steps {
        sh '''
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKERHUB_USER',
            passwordVariable: 'DOCKERHUB_TOKEN'
          )
        ]) {
          sh '''
            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${IMAGE_NAME}:latest
            docker logout
          '''
        }
      }
    }

    stage('Validate Manifests') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG')]) {
          sh '''
            kubectl version --client
            kubectl apply --dry-run=client -f postgres-deployment.yaml
            kubectl apply --dry-run=client -f postgres-service.yaml
          '''
        }
      }
    }

    stage('Render Secret') {
      steps {
        withCredentials([
          file(credentialsId: env.KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG'),
          string(credentialsId: env.POSTGRES_PASSWORD_CREDENTIAL, variable: 'POSTGRES_PASSWORD')
        ]) {
          sh '''
            kubectl create secret generic postgres-secret \
              --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" \
              --namespace ${DEPLOY_NAMESPACE} \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    stage('Deploy to Rancher Kubernetes') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG')]) {
          sh '''
            kubectl config current-context
            kubectl apply -f postgres-deployment.yaml --namespace ${DEPLOY_NAMESPACE}
            kubectl apply -f postgres-service.yaml --namespace ${DEPLOY_NAMESPACE}
            kubectl rollout status deployment/${DEPLOYMENT_NAME} --namespace ${DEPLOY_NAMESPACE}
          '''
        }
      }
    }
  }

  post {
    success {
      echo 'Postgres deployment complete.'
    }
    failure {
      echo 'Postgres deployment failed.'
    }
  }
}
