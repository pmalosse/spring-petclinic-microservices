pipeline {
    agent any
    environment {
        // This can be nexus3 or nexus2
        NEXUS_VERSION = 'nexus3'
        // This can be http or https
        NEXUS_PROTOCOL = 'http'
        // Where your Nexus is running. In my case:
        NEXUS_URL = '192.168.1.120:8081'
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = 'maven-snapshots'
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = 'nexus-credentials'
        SONARQUBE_URL = 'http://192.168.1.120'
        SONARQUBE_PORT = '9000'
    }
    stages {
        stage('SCM') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            parallel {
                stage('Compile') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            // to use the same node and workdir defined on top-level pipeline for all docker agents
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn clean compile'
                    }
                }
            }
        }
        stage('Integration Tests') {
            when {
                anyOf { branch 'master'; branch 'develop' }
            }
            agent {
                docker {
                    image 'maven:3.6.0-jdk-8-alpine'
                    args '-v /root/.m2/repository:/root/.m2/repository'
                    reuseNode true
                }
            }
            steps {
                sh 'mvn verify -Dsurefire.skip=true'
            }
            post {
                always {
                    junit '**/target/failsafe-reports/**/*.xml'
                }
                success {
                    stash(name: 'artifact', includes: '**/target/*.jar')
                    stash(name: 'pom', includes: '**/pom.xml')
                    // to add artifacts in jenkins pipeline tab (UI)
                    archiveArtifacts '**/target/*.jar'
                }
            }
        }
        stage('Code Quality Analysis') {
            parallel {
                stage('SonarQube') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            reuseNode true
                        }
                    }
                    steps {
                        sh " mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL:$SONARQUBE_PORT"
                    }
                }
            }
        }
        stage('Deploy Artifact To Nexus') {
            parallel {
                stage('Deploy admin-server') {
                    when {
                        anyOf { branch 'master'; branch 'develop' }
                    }
                    steps {
                        script {
                            list = ['spring-petclinic-admin-server', 'spring-petclinic-api-gateway', 'spring-petclinic-config-server', 'spring-petclinic-customers-service', 'spring-petclinic-discovery-server', 'spring-petclinic-vets-service']
                            for (int i = 0; i < list.size(); i++) {
                                pom = readMavenPom file: "list[i]/pom.xml"
                                filesByGlob = findFiles(glob: "$list[$i]/target/*.${pom.packaging}")
                                echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                                artifactPath = filesByGlob[0].path
                                artifactExists = fileExists artifactPath
                                if (artifactExists) {
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
                            type: pom.packaging
                            ],
                            [artifactId: pom.artifactId,
                            classifier: '',
                            file: 'pom.xml',
                            type: 'pom'
                            ]
                        ]
                        )
                        } else {
                                    error "*** File: ${artifactPath}, could not be found"
                                }
                            }
                        }
                    }
                }

            }
        }
    }
}
