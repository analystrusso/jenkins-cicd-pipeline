#!/user/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
    remote: 'https://github.com/analystrusso/jenkins-shared-library.git',
    credentialsId: 'github-creds'])
     
def gv

pipeline {   
    agent any
    tools {
        maven 'Maven'
    }
    stages {
        stage("increment version") {
            steps {
                script {
                        echo 'incrementing app version...'
                        sh 'mvn build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} versions:commit'
                        def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                        def version = matcher[0][1]
                        env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage("build jar") {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'

                }
            }
        }

        stage("build image") {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh "docker build -t analystrusso/twn-bootcamp-repo:${IMAGE_NAME} ."
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh "docker push analystrusso/twn-bootcamp-repo:${IMAGE_NAME}"
                    }
                }
            }
        }

        stage("deploy") {
            steps {
                script {
                    def dockerCmd = """
                        docker rm -f twn-bootcamp || true
                        docker run --name twn-bootcamp -p 8080:8080 -d analystrusso/twn-bootcamp-repo:${IMAGE_NAME}
                        sleep 2
                        docker ps -f name=twn-bootcamp --format "{{.Status}}"
                    """
                    sshagent(['ec2-server-key']) {
                        sh "ssh -o StrictHostKeyChecking=no ec2-user@3.80.120.211 '${dockerCmd}'"
                    }
                }
            }
        }

        stage("commit version update") {
            steps {
                script {
                    echo "pushing to github..."
                    withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'

                        sh 'git status'
                        sh 'git branch'
                        sh 'git config --list'

                        sh 'git remote set-url origin https://${USER}:${PASS}@github.com/analystrusso//jenkins-cicd-pipeline.git'
                        sh "git checkout -B ${env.BRANCH_NAME}"
                        sh "git fetch origin ${env.BRANCH_NAME}"
                        sh 'git add .'
                        sh 'git commit -m "ci:version bump"'
                        sh 'git status'
                        sh "git rebase origin/${env.BRANCH_NAME}"
                        sh "git push origin HEAD:${env.BRANCH_NAME}"
                    }
                        
                }
            }
  1      }
    }
} 
