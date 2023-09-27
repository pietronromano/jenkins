# SOURCE: https://www.jenkins.io/doc/book/pipeline/docker/#using-multiple-containers
# RESULTS: SUCCESS!
pipeline {
    agent none
    stages {
        stage('Back-end') {
            agent {
                docker { image 'maven:3.9.4-eclipse-temurin-17-alpine' }
            }
            steps {
                sh 'mvn --version'
            }
        }
        stage('Front-end') {
            agent {
                docker { image 'node:18.18.0-alpine3.18' }
            }
            steps {
                sh 'node --version'
            }
        }
    }
}