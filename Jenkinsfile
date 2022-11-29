pipeline{
    agent {
     kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          name: mypod
          namespace: default
        spec:
          serviceAccount: jenkins-agent-sa
          containers:
          - name: build-agent
            image: careem785/jenkins-build-agent:2.0
            command: 
             - cat
            tty: true
      '''
        }
    }
    stages {

  stage ('Checkout SCM'){
        steps {
          container('build-agent'){
          // checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'loginapp', url: 'https://github.com/careem111/loginapp-integration.git']]])
          git branch: 'main',    credentialsId: 'github-loginapp',    url: 'https://github.com/careem111/loginapp-integration.git'
        }
      }
   }
	  
	stage ('Build')  {
	    steps {
        container('build-agent'){
        dir('app'){
            sh "mvn package"
            }
          }
        }    
   }
   
  stage ('SonarQube Analysis') {
    steps {
      container('build-agent'){
      withSonarQubeEnv('sonar') {           
				dir('app'){
          sh 'mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar'
          }
        }
    }
    }
 }

    stage ('Artifactory configuration') {
            steps {
              container('build-agent'){
                rtServer (
                    id: "jfrog",
                    url: "https://krmkube2.jfrog.io/artifactory",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog",
                    releaseRepo: "dptweb-libs-release-local ",
                    snapshotRepo: "dptweb-libs-snapshot-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog",
                    releaseRepo: "dptweb-libs-libs-release",
                    snapshotRepo: "dptweb-libs-libs-snapshot"
                )
              }
            }
    }

    stage ('Deploy Artifacts') {
            steps {
              container('build-agent'){
                rtMavenRun (
                    tool: "maven", // Tool name from Jenkins configuration
                    pom: 'app/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
              }
         }
    }

stage('Docker Build') {
      steps {
        container('build-agent'){
            script {
            docker.withRegistry( 'https://registry.hub.docker.com', 'docker'  ) {
              dockerImage = docker.build 'careem785/logincicdapp'
              dockerImage.push('latest')
                }
              }
        }
    }
}
   stage ('Publish build info') {
            steps {
              container('build-agent'){
                rtPublishBuildInfo (
                    serverId: "jfrog"
             )
            }
        }
    }

  stage('Build Helm Charts') {
    steps {
      container('build-agent'){
        dir('charts') {
        withCredentials([usernamePassword(credentialsId: 'jfrog', usernameVariable: 'username', passwordVariable: 'password')]) {
             sh '/usr/local/bin/helm package logincicd-app'
              sh '/usr/local/bin/helm push-artifactory logincicd-app-1.0.tgz https://krmkube2.jfrog.io/artifactory/edweb-helm-local --username $username --password $password'
            }
          }
         }
        }
      }

  } 
}