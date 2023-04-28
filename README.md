### Complete CI/CD Pipeline with EKS and AWS ECR

### Technologies used:

Kubernetes, Jenkins, AWS EKS, AWS ECR, Java, Maven, Linux, Docker, Git

### Project Description:

1-Create private AWS ECR Docker repository

2-Adjust Jenkinsfile to build and push Docker Image to AWS ECR

3-Integrate deploying to K8s cluster in the CI/CD pipeline from AWS ECR private registry

4-So the complete CI/CD project we build has the following configuration:

a. CI step: Increment version

b. CI step: Build artifact for Java Maven application

c. CI step: Build and push Docker image to AWS ECR

d. CD step: Deploy new application version to EKS cluster e. CD step: Commit the version update

### Instructions:

###### Step 1: Install and configure Jenkins and docker on DigitalOcean Droplet Server

###### Step 2: Create AWS EKS cluster with Nodegroups on AWS

###### Step 3: Create a AWS ECR private repository and push image to it

###### Step 4: Install gettext-base tool for envsubst on jenkins container server

```
apt-get update
```

```
apt-get install gettext-base
```

###### Step 5: Install AWS-IAM-Authenticator

###### Step 6: Create aws-repo-credential on Jenkins by AWS ECR repo credential

###### Step 5: Create Java app deployment and service component config file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  labels:
    app: $APP_NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      imagePullSecrets:
        - name: my-registry-key
      containers:
        - name: $APP_NAME
          image: $IMAGE_REPO:$IMAGE_NAME
          imagePullPolicy: Always
          ports:
            - containerPort: 8080

```

```
apiVersion: v1
kind: Service
metadata:
  name: $APP_NAME
spec:
  selector:
    app: $APP_NAME
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

###### Step 5: Update the Deploy stage in Jenkinsfile

```
#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        ECR_REPO_URL = '664574038682.dkr.ecr.eu-west-3.amazonaws.com'
        IMAGE_REPO = "${ECR_REPO_URL}/java-maven-app"
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                    echo "${IMAGE_REPO}"
                }
            }
        }
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
                   sh 'mvn clean package'
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'ecr-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t ${IMAGE_REPO}:${IMAGE_NAME} ."
                        sh "echo $PASS | docker login -u $USER --password-stdin ${ECR_REPO_URL}"
                        sh "docker push ${IMAGE_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo 'deploying docker image...'
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'Jenkins-github-pat', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh 'git config user.email "jason2019au@gmail.com"'
                        sh 'git config user.name "jason"'
                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/DevOps-CGL/java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump${BUILD_NUMBER}"'
                        sh 'git push origin HEAD:master'
                    }
                }
            }
        }
    }
}

```
