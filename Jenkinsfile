pipeline {
        agent {
        kubernetes {
			//yaml Provide the full path with env folder
			yamlFile 'kubernetes_docker.yaml'
        }
    }
 
    stages {
        stage ('Clone') {
            steps {
                container('docker') {
                git branch: 'master', url: "https://github.com/usuryadevara/jfrog-petclinic-ci-cd.git"
                }            
            }
        }
 
        stage ('Artifactory Configuration') {
            steps {
                container('docker'){
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
        }
        
        stage('Build Maven Project') {
            steps {
                container('docker') {
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
        }
        
        stage ('Download docker client, Build Docker Image') {
            steps {
                container('docker') {
                script {
                    sh 'curl -fsSLO https://get.docker.com/builds/Linux/x86_64/docker-17.04.0-ce.tgz && tar xzvf docker-17.04.0-ce.tgz && mv docker/docker /usr/local/bin && rm -r docker docker-17.04.0-ce.tgz'
                    //docker.build("devsecopsunicloud.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                    docker.build("devsecopsunicloud.jfrog.io/docker-devsecop-docker-local/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                }
                }
            }
        }
 
        stage ('Push Image to Artifactory') {
            steps {
                container('docker') {
//                 withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jfrogadmin', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
//                     sh """
//                     mkdir ~/.docker/ && touch ~/.docker/config.json
//                     cat <<-EOF > ~/.docker/config.json
// {
//     "auths": {
//         "https://devsecopsunicloud.jfrog.io": {
//             "auth": "${env.PASSWORD}",
//             "email": "${env.USERNAME}"
//         }
//     }
// }
// EOF
//                     """
                rtDockerPush(
                    serverId: "devsecopsunicloud",
                    image: "devsecopsunicloud.jfrog.io/docker-devsecop-docker-local/" + "pet-clinic:1.0.${env.BUILD_NUMBER}",
                    targetRepo: 'docker-devsecop-docker-local',
                    properties: 'project-name=jfrog-blog-post;status=stable'
                )
                // }
                }
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
                container('docker') {
                  sh """
                    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
                    chmod 700 get_helm.sh && ./get_helm.sh && helm version
                  """
                }
            }
        }
 
        stage('Configure Helm & Artifactory') {
            steps {
                container('docker') {
                 withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jfrogadmin', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                   sh """
                    helm repo add helm https://devsecopsunicloud.jfrog.io/artifactory/devsecops-helm --username ${env.USERNAME} --password ${env.PASSWORD}
                    helm repo update
                   """
                 }
                }
            }
        }

        stage('Deploy Chart') {
            steps {
                container('docker') {
                 withCredentials([kubeconfigContent(credentialsId: 'k8s-cluster-kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
                    sh """
                     echo "$KUBECONFIG_CONTENT" > config && cp config ~/.kube/config
                     helm upgrade --install spring-petclinic-ci-cd-k8s-example helm/spring-petclinic-ci-cd-k8s-chart --kube-context=docker-desktop --set=image.tag=1.0.${env.BUILD_NUMBER}
                    """
                 }
                }
            }
        }
    }
}
