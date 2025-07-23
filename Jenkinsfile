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
"""
    }
  }

  environment {
    IMAGE_NAME = 'prj-registry.altia.es/p0010014h6/ec-template-service'
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    GIT_USER = 'miguel.sanchez.sedes'
    GIT_TOKEN = 'a17171f38c1139391637b20d2fb300b67b384c81'
    HARBOR_CREDS = credentials('altia_user_harbor')
    REPO_B = 'empresa/repo-b'
    TEMP_BRANCH = "update-image-${IMAGE_TAG}"
  }

  stages {
    stage('Clonar repo A') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: 'main']],
          userRemoteConfigs: [[
            url: "https://${GIT_USER}:${GIT_TOKEN}@garage.altia.es/Practicas-Formacion/ec-template-service.git"
          ]]
        ])
      }
    }

    stage('Construir imagen Docker y subir a Harbor') {
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

    stage('Clonar y modificar repo B') {
      steps {
        container('docker') {
          sh '''
            git clone https://${GIT_USER}:${GIT_TOKEN}@garage.altia.es/Practicas-Formacion/ec-k8s-config.git
            cd ec-k8s-config
            git checkout -b ${TEMP_BRANCH}
            sed -i "s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" deployment.yaml
            git config user.name "jenkins"
            git config user.email "jenkins@altia.es"
            git commit -am "Update image to ${IMAGE_TAG}"
            git push https://${GIT_USER}:${GIT_TOKEN}@garage.altia.es/Practicas-Formacion/ec-k8s-config.git ${TEMP_BRANCH}
          '''
        }
      }
    }
  }
}
