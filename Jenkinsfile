pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:dind
    args: ["--registry-mirror=https://mirror.gcr.io", "--storage-driver=overlay2"]
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig
  volumes:
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
    }
  }

  environment {
    REGISTRY = 'nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085'
    REPO     = 'my-repository'
    APP      = 'hello-world'
    TAG      = 'v1'
    IMAGE    = "${env.REGISTRY}/${env.REPO}/${env.APP}:${env.TAG}"
  }

  stages {
    stage('Login to Registry') {
      steps {
        container('dind') {
          sh 'docker --version'
          sh 'sleep 5'
          sh 'docker login $REGISTRY -u admin -p Changeme@2025'
        }
      }
    }

    stage('Build') {
      steps {
        container('dind') {
          sh 'docker build -t $IMAGE .'
          sh 'docker image ls $IMAGE'
        }
      }
    }

    stage('Push') {
      steps {
        container('dind') {
          sh 'docker push $IMAGE'
        }
      }
    }

    stage('Create ImagePullSecret') {
      steps {
        container('kubectl') {
          sh '''
            kubectl get ns ai-ns || kubectl create ns ai-ns
            kubectl delete secret nexus-secret -n ai-ns --ignore-not-found
            kubectl create secret docker-registry nexus-secret \
              --namespace ai-ns \
              --docker-server=$REGISTRY \
              --docker-username=admin \
              --docker-password='Changeme@2025'
          '''
        }
      }
    }

    stage('Deploy') {
      steps {
        container('kubectl') {
          sh '''
            kubectl apply -n ai-ns -f k8s/deployment.yaml
            kubectl apply -n ai-ns -f k8s/service.yaml
            kubectl rollout status -n ai-ns deploy/hello-world-deployment
          '''
        }
      }
    }
  }
}
