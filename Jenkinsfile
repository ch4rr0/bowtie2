pipeline {
  agent {
    docker {
      image 'ubuntu:latest'
    }

  }
  stages {
    stage('Build') {
      steps {
        sh '''apt-get update
apt-get install -y git libtbb-dev zlib1g-dev make g++
make allall -j4'''
      }
    }
  }
}