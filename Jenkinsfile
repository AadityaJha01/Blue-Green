pipeline {
  agent any
  environment {
    DOCKERHUB = "deku013/webapp"
    IMAGE_TAG = "latest"
    KUBECONFIG = '/var/jenkins_home/.kube/config'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build') {
      steps {
        sh "docker build -t ${DOCKERHUB}:${IMAGE_TAG} ."
      }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
          sh 'echo $DB_PASS | docker login -u $DB_USER --password-stdin'
          sh "docker push ${DOCKERHUB}:${IMAGE_TAG}"
        }
      }
    }
    stage('Deploy Blue & Service (first run)') {
      steps {
        sh 'kubectl apply -f k8s/deploy-blue.yaml'
        sh 'kubectl apply -f k8s/service.yaml'
      }
    }
    stage('Deploy Green (deploy alternate version)') {
      steps {
        sh 'kubectl apply -f k8s/deploy-green.yaml'
      }
    }
    stage('Toggle Traffic') {
      steps {
        sh '''
          CUR=$(kubectl get svc myapp-service -o jsonpath='{.spec.selector.version}')
          echo "Current -> $CUR"
          if [ "$CUR" = "blue" ]; then
            kubectl patch svc myapp-service -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'
            echo "Switched to green"
          else
            kubectl patch svc myapp-service -p '{"spec":{"selector":{"app":"myapp","version":"blue"}}}'
            echo "Switched to blue"
          fi
        '''
      }
    }
  }
  post {
    always {
      sh 'kubectl get pods -o wide || true'
      sh 'kubectl get svc || true'
    }
  }
}
