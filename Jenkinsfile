pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:20.10.17-dind
    securityContext:
      privileged: true
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
  - name: kubectl
    image: lachlanevenson/k8s-kubectl:latest
    command:
      - cat
    tty: true
"""
    }
  }
  
  environment {
    IMAGE_NAME = 'prj-registry.altia.es/p0010014h6/ec-template-service'
    GIT_CREDS = credentials('garage-rw')
    HARBOR_CREDS = credentials('altia_user_harbor')
  }

  stages {
    stage('Clone Template Service Repository') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: 'main']],
          userRemoteConfigs: [[
            url: "https://$GIT_CREDS_USR:$GIT_CREDS_PSW@garage.altia.es/Practicas-Formacion/ec-template-service.git"
          ]]
        ])

        script {
          def rawTag = sh(
            script: 'git describe --tags `git rev-list --tags --max-count=1` || echo "v0.0.0"',
            returnStdout: true
          ).trim()
          def imageTag = rawTag.replaceFirst(/^v/, '')

          env.IMAGE_TAG = imageTag
          env.TEMP_BRANCH = "feature/update-template-service-${imageTag}"

          echo "Last tag: ${rawTag} -> Using IMAGE_TAG=${imageTag}"
        }
      }
    }

    stage('Build Docker Image and Push to Harbor') {
      steps {
        container('docker') {
          sh '''
            docker version
            docker login -u $HARBOR_CREDS_USR -p $HARBOR_CREDS_PSW https://prj-registry.altia.es/
            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Clone and Modify k8s Config Repository') {
      steps {
        container('docker') {
          sh '''
            retry() {
              local max=3
              local delay=5
              local count=1
              until "$@"; do
                if [ "$count" -ge "$max" ]; then
                  echo "❌ Command failed after $count attempts: $*"
                  return 1
                fi
                echo "⚠️  Command failed (attempt $count/$max): $*"
                sleep $delay
                count=$((count + 1))
              done
            }

            retry apk add --no-cache git
            retry git clone https://$GIT_CREDS_USR:$GIT_CREDS_PSW@garage.altia.es/Practicas-Formacion/ec-k8s-config.git
            
            cd ec-k8s-config
            git checkout -b ${TEMP_BRANCH}
            sed -i "/name: ec-template-service/{n;s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|;}" ec-template-service-k8s-config/deployment.yaml
            git config user.name "jenkins"
            git config user.email "jenkins@altia.es"
            git commit -am "Update template service to ${IMAGE_TAG}"
            
            retry git push https://$GIT_CREDS_USR:$GIT_CREDS_PSW@garage.altia.es/Practicas-Formacion/ec-k8s-config.git ${TEMP_BRANCH}
          '''
        }
      }
    }

    stage('Create Pull Request') {
      steps {
        container('docker') {
          sh '''
            retry() {
              local max=3
              local delay=5
              local count=1
              until "$@"; do
                if [ "$count" -ge "$max" ]; then
                  echo "❌ Command failed after $count attempts: $*"
                  return 1
                fi
                echo "⚠️  Command failed (attempt $count/$max): $*"
                sleep $delay
                count=$((count + 1))
              done
            }

            retry apk add --no-cache curl

            retry curl -X POST \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Bearer $GIT_CREDS_PSW" \
                      -d "{\\"title\\": \\"Update template service to ${IMAGE_TAG}\\", \\"head\\": \\"${TEMP_BRANCH}\\", \\"base\\": \\"develop\\"}" \
                      https://garage.altia.es/api/v1/repos/Practicas-Formacion/ec-k8s-config/pulls
          '''
        }
      }
    }

    stage('Apply Deployment in Kubernetes') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'ec-k8s-access-config', variable: 'KUBECONFIG_FILE')]) {
            sh '''
              cd ec-k8s-config
              kubectl --kubeconfig=$KUBECONFIG_FILE apply -n form-practicas -f ec-template-service-k8s-config/deployment.yaml
            '''
          }
        }
      }
    }

  }
}
