pipeline {
  options {
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
    skipDefaultCheckout true
  }
  environment {
    HTTP_PROXY = "http://fortiproxy.dservers.dsos.gr:8888"
    HTTPS_PROXY = "http://fortiproxy.dservers.dsos.gr:8888"
    REGISTRY_REPOSITORY = "harbor.example.com/myproject/dotnet-demo"
    NEW_IMAGE_TAG = "v0.0${BUILD_NUMBER}"
    HARBOR_CREDS = credentials('harbor-credentials')
    HARBOR_CERT = credentials('harbor-certificate')
    COSIGN_KEY = credentials('cosign-key')
    COSIGN_PUBLIC = credentials('cosign-public')
    GIT_EMAIL = "automationbot@example.com"
    GIT_USERNAME = "automationbot"
    GIT_REPOSITORY = "10.24.72.55/automationbot/kubernetes-infrastructure.git"
    GIT_BRANCH = "main"
    GIT_CREDS = credentials('gitlab-credentials')
    DEPLOYMENT_FILE_PATH_BACKEND = "development/dotnet-demo/backend-dotnet-deployment.yaml"
    DEPLOYMENT_FILE_PATH_FRONTEND = "development/dotnet-demo/frontend-dotnet-deployment.yaml"
  }
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          nodeName: lgrvdkubdlaw2
          containers:
          - name: dotnet-sdk
            image: mcr.microsoft.com/dotnet/sdk:6.0
            command:
            - cat
            tty: true
            resources:
              requests:
                memory: "128Mi"
                cpu: "250m"
              limits:
                memory: "512Mi"
                cpu: "500m"
          - name: kaniko
            image: gcr.io/kaniko-project/executor:debug
            imagePullPolicy: Always
            command:
            - cat
            tty: true
            resources:
              requests:
                memory: "64Mi"
                cpu: "250m"
              limits:
                memory: "256Mi"
                cpu: "400m"
            volumeMounts:
              - name: jenkins-docker-cfg
                mountPath: /kaniko/.docker
          - name: cosign
            image: alpine:3.17.1
            command: ["/bin/sh"]
            args: ["-c", "apk update; apk add ca-certificates cosign; cat;"]
            tty: true
            resources:
              requests:
                memory: "64Mi"
                cpu: "100m"
              limits:
                memory: "128Mi"
                cpu: "150m"
          - name: git
            image: alpine/git:2.36.3
            command:
            - cat
            tty: true
            resources:
              requests:
                memory: "64Mi"
                cpu: "100m"
              limits:
                memory: "128Mi"
                cpu: "150m"
          volumes:
          - name: jenkins-docker-cfg
            projected:
              sources:
              - secret:
                  name: harbor-credentials
                  items:
                    - key: .dockerconfigjson
                      path: config.json
        '''
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh "git config --global http.sslVerify false"
        checkout scm
      }
    }
    stage('Build Application') {
      steps {
        container('dotnet-sdk') {
          sh '''
            export HTTP_PROXY=${HTTP_PROXY}
            export HTTPS_PROXY=${HTTPS_PROXY}
            
            # Build the backend app
            cd backend
            dotnet restore
            mkdir backend-app/
            dotnet publish -c release -o backend-app/
            ls -las backend-app

            # Build the frontend app
            cd ../frontend
            dotnet restore
            mkdir frontend-app/
            dotnet publish -c release -o frontend-app/
            ls -las frontend-app
          '''
        }
      }
    }
    stage('Kaniko Build & Push Image') {
      steps {
        container('kaniko') {
          sh '''
            cp \${HARBOR_CERT} /kaniko/ssl/certs/additional-ca-cert-bundle.crt
            /kaniko/executor --context backend --destination ${REGISTRY_REPOSITORY}:${NEW_IMAGE_TAG}-backend 2>&1 | tee outfile-backend.txt
            /kaniko/executor --context frontend --destination ${REGISTRY_REPOSITORY}:${NEW_IMAGE_TAG}-frontend 2>&1 | tee outfile-frontend.txt
          '''
        }
      }
    }
    stage('Sign Image & Push Signature') {
      steps {
        container('cosign') {
          sh '''
            # Trust the harbor ca certificate
            cp \${HARBOR_CERT} /usr/local/share/ca-certificates/harbor-ca.crt
            update-ca-certificates
            
            # Sign the backend image and push the signature to the registry
            cosign login harbor.example.com -u ${HARBOR_CREDS_USR} -p ${HARBOR_CREDS_PSW}
            IMAGE_URI_LATEST_DIGEST=$(tail -n 1 outfile-backend.txt | awk \'{ print $NF }\')
            export COSIGN_PASSWORD="" && cosign sign --key \${COSIGN_KEY} $IMAGE_URI_LATEST_DIGEST
            cosign verify --key \${COSIGN_PUBLIC} $IMAGE_URI_LATEST_DIGEST

            # Sign the frontend image and push the signature to the registry
            IMAGE_URI_LATEST_DIGEST=$(tail -n 1 outfile-frontend.txt | awk \'{ print $NF }\')
            export COSIGN_PASSWORD="" && cosign sign --key \${COSIGN_KEY} $IMAGE_URI_LATEST_DIGEST
            cosign verify --key \${COSIGN_PUBLIC} $IMAGE_URI_LATEST_DIGEST
          ''' 
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        container('git') {
          sh '''
            # Setup the git user and clone the repository
            mkdir temp && cd temp
            git config --global user.email ${GIT_EMAIL}
            git config --global user.name ${GIT_USERNAME}
            git config --global http.sslVerify false
            git init
            git clone https://${GIT_CREDS_USR}:${GIT_CREDS_PSW}@${GIT_REPOSITORY}
            git remote add origin https://${GIT_REPOSITORY}
            git branch -M ${GIT_BRANCH}
            cd kubernetes-infrastructure
            
            # Find the old backend image tag and replace it with the new one
            OLD_IMAGE_TAG=$(grep -E ${REGISTRY_REPOSITORY} ${DEPLOYMENT_FILE_PATH_BACKEND} | cut -d : -f 3)
            sed -i -e "s/${OLD_IMAGE_TAG}/${NEW_IMAGE_TAG}-backend/" ${DEPLOYMENT_FILE_PATH_BACKEND}
            cat ${DEPLOYMENT_FILE_PATH_BACKEND}

            # Find the old frontend image tag and replace it with the new one
            OLD_IMAGE_TAG=$(grep -E ${REGISTRY_REPOSITORY} ${DEPLOYMENT_FILE_PATH_FRONTEND} | cut -d : -f 3)
            sed -i -e "s/${OLD_IMAGE_TAG}/${NEW_IMAGE_TAG}-frontend/" ${DEPLOYMENT_FILE_PATH_FRONTEND}
            cat ${DEPLOYMENT_FILE_PATH_FRONTEND}
            
            # Commit the changes to the git repository
            git add .
            git commit -m "Update dotnet-demo-app version to ${NEW_IMAGE_TAG}"
            git push -uf origin ${GIT_BRANCH} 
 
          '''
        }
      }
    }
  }
}
