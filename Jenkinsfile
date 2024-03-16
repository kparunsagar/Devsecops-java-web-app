pipeline {
  agent { label 'docker-build-node' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
  }
  tools{
      jdk 'jdk17'
      maven 'maven'
  }
  environment {
      SCANNER_HOME=tool 'sonarQube'
      DOCKER_TOKEN: ${{ env.DOCKER_TOKEN }}
      DOCKERHUB_CREDENTIALS = credentials('docker')
      CI = true
      ARTIFACTORY_ACCESS_TOKEN = credentials('artifactory-access-token')
  }
  stages {
    stage("Git Checkout"){
      steps{
        git branch: 'main', changelog: false, poll: false, url: 'https://github.com/kparunsagar/java-web-app.git'
      }
    }
            
    stage("Compile"){
      steps{
        sh "mvn clean compile"
      }
    }
    stage("Test Cases"){
      steps{
        sh "mvn test"
      }
    }
    stage('Build') {
      steps {
        sh './mvnw clean install'
      }
    }
    stage("OWASP Dependency Check"){
      steps{
        dependencyCheck additionalArguments: '', odcInstallation: 'DP'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    stage("Sonarqube Analysis "){
      steps{
        withSonarQubeEnv('sonarQube') {
          sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=javawebapp \
          -Dsonar.java.binaries=. \
          -Dsonar.projectKey=javawebapp '''
        }
      }
    }
    stage('Packaging') {
      steps {
        step([$class: 'ArtifactArchiver', artifacts: '**/target/*.jar', fingerprint: true])
      }
    }         
    stage ("Artifactory Publish"){
      steps {
        script{
          def server = Artifactory.server 'Artifactory'
          def rtMaven = Artifactory.newMavenBuild()
          //rtMaven.resolver server: server, releaseRepo: 'jenkinsdemo_repo', snapshotRepo: 'demopipeline'
          rtMaven.deployer server: server, releaseRepo: 'javaproject-javarelease-snapshot', snapshotRepo: 'javaproject-javaprojectlocal-snapshot'
          rtMaven.tool = 'maven'
                            
          def buildInfo = rtMaven.run pom: '$workspace/pom.xml', goals: 'clean install'
          rtMaven.deployer.deployArtifacts = true
          rtMaven.deployer.deployArtifacts buildInfo

          server.publishBuildInfo buildInfo
        }
      }
    }

    stage("Build docker image"){
      steps {
        sh 'docker build -t kparun/javawebapp:$BUILD_NUMBER .'
      }
    }
    stage("Login to docker hub"){
      steps {
        sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
      }
    }
    stage("Docker push"){
      steps {
        sh 'docker push kparun/javawebapp:$BUILD_NUMBER'
      }
    }
  }
}
