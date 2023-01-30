# Jenkins & Kubernetes 

## Scripts 
Before getting started, I installed a number of tools using scripts within this repository: 
* install-cli: installs the amazon web services (aws) command line tool (cli).
* install-kubectl: installs the kubernetes cli.
* install-eksctl: installs a simple CLI tool for creating and managing clusters on EKS - Amazon's managed Kubernetes service for EC2.
* install-jenkins: installs jenkins, an open source automation server which enables developers around the world to reliably build, test, and deploy their software.

Also, another script was created to create a cluster on our EC2 instance. The create-cluster.sh files contents:
```
eksctl create cluster \
--name adams-cluster \
--region eu-west-1 \
--nodegroup-name adams-nodes \
--nodes 2 \
--nodes-min 2 \
--nodes-max 10 \
--node-type t3.micro \
--with-oidc \
--ssh-access \
--ssh-public-key AMSKey \
--managed
```

## Backend 
The back.yml files contents:
```
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP     # set the type of Service
  selector:
    app: backend      # referencing the Pod's label
  ports:
  - protocol: TCP
    port: 5001
    targetPort: 5001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend      # needs to be the same as Service's selector
spec:
  selector:
    matchLabels:
      app: backend
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 33%
      maxUnavailable: 33%
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: htrvolker/python-backend:letters
        ports:
        - containerPort: 5001
```
Above we are defining two manifests: one for the ClusterIP Service, and one for our backend Python server, which is running as a single Pod. This will expose the backend to the cluster's internal network, allowing the frontend Pod to send HTTP requests to it. The frontend app is written to make requests to a network location called backend on port 5001, so it is important that the Service reflects this.

## Frontend 
The front.yml files contents:
```
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer  # set the type of Service
  selector:
    app: frontend     # referencing the Pod's label
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend     # needs to be the same as Service's selector
spec:
  selector:
    matchLabels:
      app: frontend
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 33%
      maxUnavailable: 33%
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: htrvolker/python-frontend:red
        ports:
        - containerPort: 5000
```

Above we are defining two manifests: one for the LoadBalancer Service, and one for our frontend Python server, which is running as a single Pod. This will expose the frontend to the internet via a load balancer, allowing you to connect to it on your browser. The selector: mapping under spec: in the Service manifest is referencing the app: frontend label found under the Pod's metadata: mapping. This is how the Service becomes associated with the Pod.

## Jenkinsfile
A Jenkinsfile was created with a number of stages:
1. install the necessary tools: aws cli, kubernetes cli and the eks cli. 
2. configure the kubernetes cli.
3. creates a namespace.
4. apply the manifests.

The Jenkinsfile contents:
```
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
```




