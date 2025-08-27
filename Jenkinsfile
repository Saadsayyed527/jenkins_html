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
    args: ["--insecure-registry=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085", "--storage-driver=overlay2"]
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
    TAG      = "build-${env.BUILD_NUMBER}"   // unique tag each build
    IMAGE    = "${env.REGISTRY}/${env.REPO}/${env.APP}:${env.TAG}"
  }

  stages {
    stage('Login to Registry') {
      steps {
        container('dind') {
          sh 'docker login $REGISTRY -u admin -p Changeme@2025'
        }
      }
    }

    stage('Build & Push') {
      steps {
        container('dind') {
          sh '''
            echo "🚀 Building Docker image..."
            docker build -t $IMAGE .
            docker push $IMAGE
          '''
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
            echo "🚀 Updating Kubernetes Deployment..."
            kubectl set image deployment/hello-world-deployment \
              hello-world=$IMAGE -n ai-ns --record

            echo "🔍 Checking rollout status..."
            kubectl rollout status deployment/hello-world-deployment -n ai-ns --timeout=60s || {
              echo "❌ Rollout failed, showing debug info..."
              kubectl describe deployment hello-world-deployment -n ai-ns
              kubectl get pods -n ai-ns -l app=hello-world -o wide
              kubectl logs -n ai-ns -l app=hello-world --tail=50
              exit 1
            }
          '''
        }
      }
    }
  }
}
