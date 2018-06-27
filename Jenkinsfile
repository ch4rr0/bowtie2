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
git clone https://github.com/ch4rr0/bowtie2.git
cd bowtie2
make allall -j4'''
      }
    }
  }
}