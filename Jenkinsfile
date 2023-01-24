pipeline {
  environment {
    NAMESPACE = "development"
  }
  agent any
  stages {

    stage('Install CLIs') {
      steps {
        sh "sudo chmod +x *.sh"
        sh "./install-cli.sh"
        sh "./install-eksctl.sh"
        sh "./install-kubectl.sh"
      }
    }

    stage('Configure kubectl') {
      steps {
        sh "aws eks --region eu-west-1 update-kubeconfig --name adams-cluster"
        sh "kubectl config set-context --current --namespace=${development}"
      }
    }

    stage('Create Namespaces'){
      steps {
        sh "kubectl create ns development" // any changes to dev branch will happen here
      }
    }

    //

    stage('Deploy manifests') {
      steps {
        sh "kubectl apply -f back.yml"
        sh "kubectl apply -f front.yml"
        sh "kubectl get services"
      }
    }
  }
}

