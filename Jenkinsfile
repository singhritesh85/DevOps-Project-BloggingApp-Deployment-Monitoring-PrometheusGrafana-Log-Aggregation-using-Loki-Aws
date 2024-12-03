pipeline{
    agent{
        node{
            label "Slave-1"
            customWorkspace "/home/jenkins/dexter"
        }
    }
    environment{
        JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        PATH="$PATH:$JAVA_HOME/bin:/opt/apache-maven/bin:/opt/node-v16.0.0/bin:/usr/local/bin"
    }
    parameters { 
        string(name: 'COMMIT_ID', defaultValue: '', description: 'Provide the Commit ID') 
        string(name: 'REPO_NAME', defaultValue: '', description: 'Provide the ECR Repository Name for Application Image')
        string(name: 'TAG_NAME', defaultValue: '', description: 'Provide the TAG Name')
        string(name: 'REPLICA_COUNT', defaultValue: '', description: 'Provide the number of Pods to be created')
    }
    stages{
        stage("Clone-Code"){
            steps{
                checkout scmGit(branches: [[name: '${COMMIT_ID}']], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-cred', url: 'https://github.com/singhritesh85/Blogging-App.git']])
            }
        }
        stage("SonarQube Analysis and Build"){
            steps{
                withSonarQubeEnv('SonarQube-Server') {
                    sh 'mvn clean install sonar:sonar'
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage("Nexus-Artifactory-Upload"){
            steps{
                script{
                    def mavenPom = readMavenPom file: 'pom.xml'
                    def nexusRepository = mavenPom.version.endsWith("SNAPSHOT") ? "maven-snapshot" : "maven-release"
                    nexusArtifactUploader artifacts: [[artifactId: 'twitter-app', classifier: '', file: "target/twitter-app-${mavenPom.version}.jar", type: 'jar']], credentialsId: 'nexus', groupId: 'com.example', nexusUrl: 'nexus.singhritesh85.com', nexusVersion: 'nexus3', protocol: 'https', repository: "${nexusRepository}", version: "${mavenPom.version}"
                }
            }
        }
        stage("Trivy Filesystem Scan"){
            steps{
                sh 'trivy fs . > /home/jenkins/trivyfilescan.txt'
            }
        }
        stage("Docker Image Scan"){
            steps{
                sh 'docker build -t ${REPO_NAME}:${TAG_NAME} .'
                sh 'trivy image --exit-code 0 --severity MEDIUM,HIGH ${REPO_NAME}:${TAG_NAME}'
                //sh 'trivy image  --exit-code 1 --severity CRITICAL ${REPO_NAME}:${TAG_NAME}'
                sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 027330342406.dkr.ecr.us-east-2.amazonaws.com'
                sh 'docker push ${REPO_NAME}:${TAG_NAME}'
            }
        }
        stage("Kubernetes Deployment"){
            steps{
                //sh 'yes|argocd login argocd.singhritesh85.com --username admin --password Admin@123'
                sh 'argocd login argocd.singhritesh85.com --username admin --password Admin@123 --skip-test-tls  --grpc-web'
                sh 'argocd app create blogapp --project default --repo https://github.com/singhritesh85/helm-repo-for-ArgoCD.git --path ./folo --dest-namespace blogapp --dest-server https://kubernetes.default.svc --helm-set service.port=80 --helm-set image.repository=${REPO_NAME} --helm-set image.tag=${TAG_NAME} --helm-set replicaCount=${REPLICA_COUNT} --upsert'
                sh 'argocd app sync blogapp'
            }
        }
    }
}
