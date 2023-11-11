pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            app: test
        spec:
          containers:
          - name: maven
            image: maven:3.8.3-adoptopenjdk-11
            command:
            - cat
            tty: true
            volumeMounts:
            - mountPath: "/root/.m2/repository"
              name: cache
          - name: git
            image: bitnami/git:latest
            command:
            - cat
            tty: true
          - name: docker
            image: docker:latest
            command:
            - cat
            tty: true
            volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-sock
          - name: sonarcli
            image: sonarsource/sonar-scanner-cli:latest
            command:
            - cat
            tty: true
          - name: kubectl-helm-cli
            image: kunchalavikram/kubectl_helm_cli:latest
            command:
            - cat
            tty: true
          - name: curl
            image: alpine/curl:latest
            command:
            - cat
            tty: true
          volumes:
          - name: cache
            persistentVolumeClaim:
              claimName: maven-cache
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
      '''
    }      
  }
  environment{
    NEXUS_VERSION = "nexus3"
    NEXUS_PROTOCOL = "http"
    NEXUS_URL = "IP:PORT"
    NEXUS_REPOSITORY = "maven-hosted"
    NEXUS_CREDENTIAL_ID = "CREDS-ID"
    DOCKERHUB_USERNAME = "USERNAME"
    APP_NAME = "spring-petclinic"
    IMAGE_NAME = "${DOCKERHUB_USERNAME}" + "/" + "${APP_NAME}"
    IMAGE_TAG = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout SCM') {
      when { expression { true } }
      steps {
        container('git') {
          git url: 'https://github.com/kunchalavikram1427/spring-petclinic.git',
          branch: 'main'
        }
      }
      post {
        success {
          sendStatus("Git Checkout","success")
        }
      }
    }
    stage('Build SW'){
      when { expression { true } }
      steps {
        container('maven'){
          sh 'mvn -Dmaven.test.failure.ignore=true clean package'
        }
      }
      post {
        success {
          junit '**/target/surefire-reports/*.xml'
          sendStatus("SW Build","success")
        }
      }
    }
    stage('Sonar Scan'){
      when { expression { true } }
      steps{
        container('sonarcli'){
          withSonarQubeEnv(credentialsId: 'CREDS', installationName: 'SERVER_DETAILS') { 
            sh '''/opt/sonar-scanner/bin/sonar-scanner \
              -Dsonar.projectKey=petclinic \
              -Dsonar.projectName=petclinic \
              -Dsonar.projectVersion=1.0 \
              -Dsonar.sources=src/main \
              -Dsonar.tests=src/test \
              -Dsonar.java.binaries=target/classes  \
              -Dsonar.language=java \
              -Dsonar.sourceEncoding=UTF-8 \
              -Dsonar.java.libraries=target/classes
            '''
          }
        }
      }
      post {
        success {
          sendStatus("Sonar Scan","success")
        }
      }
    }
    stage('Wait for Quality Gate'){
      when { expression { true } }
      steps{
        container('sonarcli'){
          timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
          }
        }
      }
      post {
        success {
          sendStatus("QG Check","success")
        }
      }
    }
    stage('Publish Maven Artifacts to Nexus'){
      when { expression { true } }
      steps {
        container('jnlp'){
          script {
            pom = readMavenPom file: "pom.xml";
            filesByGlob = findFiles(glob: "target/*.${pom.packaging}"); 
            echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
            artifactPath = filesByGlob[0].path;
            artifactExists = fileExists artifactPath;
            if(artifactExists) {
                echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                nexusArtifactUploader(
                    nexusVersion: NEXUS_VERSION,
                    protocol: NEXUS_PROTOCOL,
                    nexusUrl: NEXUS_URL,
                    groupId: pom.groupId,
                    version: pom.version,
                    repository: NEXUS_REPOSITORY,
                    credentialsId: NEXUS_CREDENTIAL_ID,
                    artifacts: [
                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: artifactPath,
                        type: pom.packaging],

                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: "pom.xml",
                        type: "pom"]
                    ]
                );

            } else {
                error "*** File: ${artifactPath}, could not be found";
            }
          }
        }
      }
      post {
        success {
          sendStatus("Push to Nexus","success")
        }
      }
    }
    stage('Publish Maven Artifacts using CURL'){
      when { expression { false } }
      steps{
        container('curl') {
          script {
            pom = readMavenPom file: "pom.xml";
            withCredentials([usernamePassword(credentialsId: 'CREDS', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
              sh "curl -v -u $USER:$PASS --upload-file target/${pom.artifactId}-${pom.version}.${pom.packaging} \
              http://NEXUS-SERVER-DETAILS/repository/REPO-NAME/org/springframework/samples/${pom.artifactId}/${pom.version}/${pom.artifactId}-${pom.version}.${pom.packaging}"
            }
          }
        }
      }
    }
    stage('Build Docker Image'){
      when { expression { true } }
      steps{
        container('docker'){
          sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
          sh "docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest"
          withCredentials([usernamePassword(credentialsId: 'CREDS', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
             sh "docker login -u $USER -p $PASS"
             sh "docker push $IMAGE_NAME:$IMAGE_TAG"
             sh "docker push $IMAGE_NAME:latest"
          }
          sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
          sh "docker rmi $IMAGE_NAME:latest"
        }
      }
      post {
        success {
          sendStatus("Push to Dockerhub","success")
        }
      }
    }
    stage("Deploy to Kubernetes via manifests"){
      when { expression { false } }
      steps{
        container('kubectl-helm-cli'){
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'CONFIG-FILE', namespace: '', serverUrl: '') {
            sh "kubectl apply -f deployment.yml"
          }
        }
      }
    }
    stage("Deploy to Kubernetes via Helm"){
      when { expression { false } }
      steps{
        container('kubectl-helm-cli'){
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'CONFIG-FILE', namespace: '', serverUrl: '') {
            sh "helm upgrade --install petclinic petclinic-chart/"
          }
        }
      }
      post {
        success {
          sendStatus("Deployment","success")
        }
      }
    }
    stage("Deploy to Kubernetes via Helm from Nexus"){
      when { expression { true } }
      steps{
        container('kubectl-helm-cli'){
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', serverUrl: '') {
            sh "helm repo add helm-hosted http://USER:TOKEN@NEXUS-SERVER/repository/HELM-REPO/"
            sh "helm repo update"
            sh "helm upgrade --install test helm-hosted/petclinic-chart"
          }
        }
      }
      post {
        success {
          sendStatus("Deployment","success")
        }
      }
    }
  }
  post {
    failure {
      mail to: 'vikram@gmail.com',
      from: 'jenkinsadmin@gmail.com',
      subject: "Jenkins pipeline has failed for job ${env.JOB_NAME}",
      body: "Check build logs at ${env.BUILD_URL}"
    }
    success {
      mail to: 'vikram@gmail.com',
      from: 'jenkinsadmin@gmail.com',
      subject: "Jenkins pipeline for job ${env.JOB_NAME} is completed successfully",
      body: "Check build logs at ${env.BUILD_URL}"
    }
  }
}       

void sendStatus(String stage, String status) {
    container('curl') {
        withCredentials([string(credentialsId: 'TOKEN-CREDS-NAME', variable: 'TOKEN')]) {
            sh "curl -u USER-NAME:$TOKEN -X POST 'https://api.github.com/repos/kunchalavikram1427/spring-petclinic/statuses/$SHA_ID' -H 'Accept: application/vnd.github.v3+json' -d '{\"state\": \"$status\",\"context\": \"$stage\", \"description\": \"Jenkins\", \"target_url\": \"$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/console\"}' "
        }
    }
}