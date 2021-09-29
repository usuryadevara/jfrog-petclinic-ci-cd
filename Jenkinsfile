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
                git branch: 'to_remove', url: "https://github.com/usuryadevara/jfrog-petclinic-ci-cd.git"
                }            
            }
        }
 
        stage ('Artifactory Configuration') {
            steps {
                container('docker'){
                rtServer (
                    id: "devsecopsunicloud",
                    url: "https://devsecopsunicloud.jfrog.io/artifactory",
                    credentialsId: "jfrogadmin"
                )
                }
            }
        }
        
        stage ('Download docker client, Build Docker Image') {
            steps {
                container('docker') {
                script {
                    sh 'curl -fsSLO https://get.docker.com/builds/Linux/x86_64/docker-17.04.0-ce.tgz && tar xzvf docker-17.04.0-ce.tgz && mv docker/docker /usr/local/bin && rm -r docker docker-17.04.0-ce.tgz'
                    docker.build("devsecopsunicloud.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                }
                }
            }
        }
 
        stage ('Push Image to Artifactory') {
            steps {
                container('docker') {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jfrogadmin', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                    sh """
                    mkdir ~/.docker/ && touch ~/.docker/config.json
                    cat << EOF > ~/.docker/config.json
                    {
                        "auths": {
                            "https://devsecopsunicloud.jfrog.io": {
                                "auth": ${env.PASSWORD},
                                "email": "${env.USERNAME}"
                            }
                        }
                    }
                    EOF
                    sleep 6000
                    """
                rtDockerPush(
                    serverId: "devsecopsunicloud",
                    image: "devsecopsunicloud.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}",
                    targetRepo: 'docker',
                    properties: 'project-name=jfrog-blog-post;status=stable'
                )
                }
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
    }
}
