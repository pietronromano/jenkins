# DATE: 29-09-2023
## Tutorial CONTINUED FROM jenkins-vm-with-agents.md / sonarqube.md
## SOURCE: https://github.com/pietronromano/complete-prodcution-e2e-pipeline
## RESULT: SUCCESS! Creates images, uploads to:
    https://hub.docker.com/repository/docker/pietronromano/complete-prodcution-e2e-pipeline/general

# REFERENCES:
- https://youtu.be/q4g7KJdFSn0?t=5390

# Add Docker Plugins to Jenkins Controller
- Docker Commons	 
- Docker Pipeline	 
- Docker API	 
- Docker	
- docker-build-step
- CloudBees Docker Build and Publish

# Create Credential
## Dockerhub: create Access Token
- https://hub.docker.com/ pietronromano axml-xsl0
- Account Settings -> Access Token
  - ID: jenkins-token: dckr_pat_z5mKJ-eaTyCL5b_nwYUUAE_7BaE

## Jenkins Credential
- Manage Jenkins -> Credentials New Credential -> Username and Password
  - Paste in access token
  - ID: dockerhub-token

# Update Pipeline
      environment {
        APP_NAME = "complete-prodcution-e2e-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "pietronromano"
        DOCKER_PASS = 'dockerhub-token'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }

        }

# Create Dockerfile
    FROM maven:3.9.0-eclipse-temurin-17 as build
    WORKDIR /app
    COPY . .
    RUN mvn clean install

    FROM eclipse-temurin:17.0.6_10-jdk
    WORKDIR /app
    COPY --from=build /app/target/demoapp.jar /app/
    EXPOSE 8080
    CMD ["java", "-jar","demoapp.jar"]

# Test image on Agent
    ssh jenkins@13.95.168.237
    docker container run --name demoapp -d -p 8080:8080 pietronromano/complete-prodcution-e2e-pipeline

    docker container exec -it demoapp sh

