pipeline {
  agent {
    docker {
      image 'ubuntu:latest'
      args '--privileged --pid=host'
    }

  }
  stages {
    stage('Build') {
      steps {
        sh '''apt-get update
apt-get install -y git libtbb-dev zlib1g-dev make g++
git pull
make allall -j4'''
      }
    }
  }
}