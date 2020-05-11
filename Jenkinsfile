String registry = "clarksource"
String repository = "k8t"
String image "${registry}/${repository}"

def dockerImage = null

pipeline {
  agent {
    kubernetes {
      defaultContainer 'python'
      yamlFile '.jenkins/python.yaml'
    }
  }

  options {
    timeout(time: 2, unit: 'HOURS')
    buildDiscarder(
      logRotator(
        daysToKeepStr: '14',
        numToKeepStr: '100',
        artifactDaysToKeepStr: '30',
      ))
  }

  stages {
    stage('setup') {
      steps {
        sh 'apk add git gcc musl-dev linux-headers libffi-dev libressl-dev'
        sh 'pip install --upgrade tox'
      }
    }

    stage('test') {
      steps {
        sh 'tox'
      }
    }

    stage('build') {
      when {
        anyOf {
          branch 'master'
          buildingTag()
        }
      }

      steps {
        sh 'pip install --upgrade setuptools wheel'
        sh 'python3 setup.py sdist bdist_wheel'
      }
    }

    stage('release') {
      when {
        buildingTag()
      }

      steps {
        sh 'pip install --upgrade twine'
        withCredentials([usernamePassword(credentialsId: 'pypi', usernameVariable: 'TWINE_USERNAME', passwordVariable: 'TWINE_PASSWORD')]) {
          sh 'twine upload dist/*'
        }
      }
    }

    stage('docker build') {
      steps {
        script {
          dockerImage = docker.build("${image}:${env.GIT_COMMIT}")
        }
      }
    }

    stage('docker build') {
      when {
        buildingTag()
      }

      steps {
        script {
          docker.withRegistry(credentialsId: 'dockerhub') {
            dockerImage.push(env.TAG_NAME)
            dockerImage.push("latest")
          }
        }
      }
    }
  }

  post {
    always{
      script {
        try {
          sh "docker rmi ${image}:${env.GIT_COMMIT}"
        } finally {
          deleteDir()
        }
      }
    }
  }
}
