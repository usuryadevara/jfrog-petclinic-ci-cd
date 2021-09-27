pipeline {
    agent any
 
    stages {
        stage ('Clone') {
            steps {
                git branch: 'master', url: "https://github.com/usuryadevara/jfrog-petclinic-ci-cd.git"
            }
        }
 
        stage ('Artifactory Configuration') {
            steps {
                rtServer (
                    id: "devsecopsunicloud",
                    url: "https://devsecopsunicloud.jfrog.io/artifactory",
                    credentialsId: "artifactory-access-token"
                )
 
                rtMavenResolver (
                    id: 'maven-resolver',
                    serverId: 'devsecopsunicloud',
                    releaseRepo: 'devsecopsunicloud-libs-release',
                    snapshotRepo: 'devsecopsunicloud-libs-snapshot'
                )  
                 
                rtMavenDeployer (
                    id: 'maven-deployer',
                    serverId: 'devsecopsunicloud',
                    releaseRepo: 'devsecopsunicloud-libs-release-local',
                    snapshotRepo: 'devsecopsunicloud-libs-snapshot-local',
                    threads: 6,
                    properties: ['BinaryPurpose=Technical-BlogPost', 'Team=DevOps-Acceleration']
                )
            }
        }
        
        stage('Build Maven Project') {
            steps {
                rtMavenRun (
                    tool: 'maven',
                    pom: 'pom.xml',
                    goals: '-U clean install',
                    opts: '-Xms1024m -Xmx4096m',
                    deployerId: "maven-deployer",
                    resolverId: "maven-resolver"
                )
            }
        }
 
        stage ('Build Docker Image') {
            steps {
                script {
                    docker.build("devsecopsunicloud.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                }
            }
        }
 
        stage ('Push Image to Artifactory') {
            steps {
                rtDockerPush(
                    serverId: "devsecopsunicloud",
                    image: "devsecopsunicloud.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}",
                    targetRepo: 'docker',
                    properties: 'project-name=jfrog-blog-post;status=stable'
                )
            }
        }
 
        stage ('Publish Build Info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "devsecopsunicloud"
                )
            }
        }
 
        stage('Install Helm') {
            steps {
                  sh """
                    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
                    chmod 700 get_helm.sh && helm version
                  """
            }
        }
 
        stage('Configure Helm & Artifactory') {
            steps {
                 withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'artifactory-access-token', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                   sh """
                    helm repo add helm https://devsecopsunicloud.jfrog.io/artifactory/helm --username ${env.USERNAME} --password ${env.PASSWORD}
                    helm repo update
                   """
                 }
            }
        }
 
        stage('Deploy Chart') {
            steps {
                withCredentials([kubeconfigContent(credentialsId: 'k8s-cluster-kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
                    sh """
                     echo "$KUBECONFIG_CONTENT" > config && cp config ~/.kube/config
                     helm upgrade --install spring-petclinic-ci-cd-k8s-example helm/spring-petclinic-ci-cd-k8s-chart --kube-context=gke_dev_us-west1-a_artifactory --set=image.tag=1.0.${env.BUILD_NUMBER}
                    """
                }
            }
        }
    }
}
