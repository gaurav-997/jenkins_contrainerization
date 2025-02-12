pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: build-${UUID.randomUUID().toString()}
spec:
  containers:
  - name: build
    image: dpthub/dpt10jenkinsagent
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1024Mi"
        cpu: "1000m"
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
// The pod is dynamically provisioned with the required image.
// Used UUID for unique and dynamic labels in the pod configuration. Each pipeline run will have a unique pod label.
        }
    }
    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'master', description: 'Branch to build')
        string(name: 'REPO_URL', defaultValue: 'https://dptrealtime@bitbucket.org/dptrealtime/eos-micro-services-admin-source.git', description: 'Git repository URL')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    }
// Added environment block for centralized management of credentials and other configuration variables (e.g., IMAGE_TAG).
// All sensitive credentials (Git, JFrog, Docker) are securely accessed using Jenkins credentials (credentialsId).
    environment {
        GIT_CREDENTIALS = credentials('git')             // Git credentials ID
        JFROG_CREDENTIALS = credentials('jfrog')         // JFrog credentials ID
        DOCKER_CREDENTIALS = credentials('docker')       // Docker Hub credentials ID
        DYNAMIC_LABEL = "build-${UUID.randomUUID().toString()}" // Dynamic label
    }
    options {
        timestamps()                                      // Add timestamps to logs for better debugging 
        ansiColor('xterm')                                 // Improve log readability in the Jenkins Console
    }
    stages {
        stage('Checkout SCM') {
            steps {
                git branch: "${params.BRANCH_NAME}", 
                    url: "${params.REPO_URL}", 
                    credentialsId: "${GIT_CREDENTIALS}"
            }
        }
// Added try-catch blocks in all critical stages, Ensures any stage failure provides a meaningful error message.
        stage('Build Maven Project') {
            steps {
                container('build') {  // it ensures that below steps should run inside a container name build 
                    script {
                        try {
                            sh './mvnw clean package'
                        } catch (Exception e) {
                            error("Maven build failed: ${e}")
                        }
                    }
                }
            }
        }
        stage('Artifactory Configuration') {
            steps {
                container('build') {
                    script {
                        try {
                            rtServer(
                                id: "jfrog",
                                url: "https://dpt13.jfrog.io/artifactory",
                                credentialsId: "${JFROG_CREDENTIALS}"
                            )
                            rtMavenDeployer(
                                id: "MAVEN_DEPLOYER",
                                serverId: "jfrog",
                                releaseRepo: "dpt13-libs-release-local",
                                snapshotRepo: "dpt13-libs-snapshot-local"
                            )
                            rtMavenResolver(
                                id: "MAVEN_RESOLVER",
                                serverId: "jfrog",
                                releaseRepo: "dpt13-libs-release",
                                snapshotRepo: "dpt13-libs-snapshot"
                            )
                        } catch (Exception e) {
                            error("Artifactory configuration failed: ${e}")
                        }
                    }
                }
            }
        }
        stage('Deploy Artifacts to Artifactory') {
            steps {
                container('build') {
                    script {
                        try {
                            rtMavenRun(
                                useWrapper: true,
                                pom: 'pom.xml',
                                goals: 'clean install',
                                deployerId: "MAVEN_DEPLOYER",
                                resolverId: "MAVEN_RESOLVER"
                            )
                        } catch (Exception e) {
                            error("Artifact deployment failed: ${e}")
                        }
                    }
                }
            }
        }
        stage('Docker Build and Push') {
            steps {
                container('build') {
                    script {
                        try {
                            docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS}") {
                                def customImage = docker.build("dpthub/eos-micro-services-admin:${params.IMAGE_TAG}")
                                customImage.push()
                            }
                        } catch (Exception e) {
                            error("Docker build or push failed: ${e}")
                        }
                    }
                }
            }
        }
        stage('Helm Chart') {
            steps {
                container('build') {
                    dir('charts') {
                        script {
                            try {
                                withCredentials([usernamePassword(credentialsId: "${JFROG_CREDENTIALS}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                                    sh '/usr/local/bin/helm package micro-services-admin'
                                    sh """
                                        /usr/local/bin/helm push-artifactory micro-services-admin-1.0.tgz \
                                        https://dpt13.jfrog.io/artifactory/dpt13-helm-local \
                                        --username $USERNAME --password $PASSWORD
                                    """
                                }
                            } catch (Exception e) {
                                error("Helm chart packaging or deployment failed: ${e}")
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                notifySlack("SUCCESS")
            }
        }
        failure {
            script {
                notifySlack("FAILURE")
            }
        }
    }
}
//Added post block to send Slack notifications based on the build status (SUCCESS or FAILURE).
// Centralized Slack notification logic via a helper function (notifySlack).
void notifySlack(String status) {
    def message = "Build ${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
    def color = (status == 'SUCCESS') ? 'good' : 'danger'
    slackSend(
        channel: '#devops',  // Replace with your Slack channel
        color: color,
        message: message
    )
}
