# Jenkins CICD Pipeline

This project is a multi-branch pipeline in Jenkins. A webhook from Github to Jenkins triggers a pipeline build upon pushing changes to Git. Then the app version is incremented, the app is built, the docker image is build and published to Docker Hub, the app is deployed on a preexisting EC2 instance with Docker preinstalled, and the incremented app and image version get pushed back to Git so it updates in the repo.
