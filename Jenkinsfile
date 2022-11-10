pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    }
        stages{
            stage("Sonar quality check"){
                agent {
                    docker {
                        image 'openjdk:11'
                    }
                }
                steps{
                    script{
                        withSonarQubeEnv(credentialsId: 'sonartoken') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                        }

                         timeout(time: 1, unit: 'HOURS') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
            stage("docker build and docker push"){
                steps{
                    script{
                        withCredentials([string(credentialsId: 'docker_pass_nexus', variable: 'docker_password')]) {
                                        sh '''
                                        docker build -t 34.125.208.211:8083/springapp:${VERSION} .
                                        docker login -u admin -p $docker_password 34.125.208.211:8083
                                        docker push 34.125.208.211:8083/springapp:${VERSION}
                                        docker rmi 34.125.208.211:8083/springapp:${VERSION}
                                        '''
                        }
                        
                    }

                }
            }
            stage("identifying misconfigs using datree in helmcharts"){
                steps{
                    script{
                        dir('kubernetes/') {
                            withEnv(["DATREE_TOKEN=${DATREE_TOKEN}"]) {
                                sh 'helm datree test myapp/'
                                }
                            }
                    }
                }
            }
            stage("pushing helm charts to nexus"){
                steps{
                    script{
                        withCredentials([string(credentialsId: 'docker_pass_nexus', variable: 'docker_password')]) {
                                        dir('kubernetes/') {
                                                sh '''
                                                    helmversion=$(helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                                    tar -czvf myapp-${helmversion}.tgz myapp/
                                                    curl -u admin:$docker_password http://34.125.208.211:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                                                '''
                                        }
                                }
                        
                        }

                    }
                }
            }
        
        post{
            always{
                echo "SUCCESS"
            }
        }


}