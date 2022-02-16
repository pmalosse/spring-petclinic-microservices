pipeline {
    agent any
    environment {
        // This can be nexus3 or nexus2
        NEXUS_VERSION = 'nexus3'
        // This can be http or https
        NEXUS_PROTOCOL = 'http'
        // Where your Nexus is running. In my case:
        NEXUS_URL = 'localhost:8081'
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = 'petclinic'
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = 'nexus-credentials'

        SONARQUBE_URL = 'http://loclahost'
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
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn clean compile'
                    }
                }
                stage('CheckStyle') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn checkstyle:checkstyle'
                    }
                    post {
                        always {
                            recordIssues enabledForFailure: true, tool: checkStyle()
                        }
                    }
                }
            }
        }
        stage('Unit Tests') {
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
                sh ' mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
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
                 junit allowEmptyResults: true, testResults: 'target/failsafe-reports/**/*.xml'
                }
                success {
                    stash(name: 'artifact', includes: 'target/*.jar')
                    stash(name: 'pom', includes: 'pom.xml')
                    archiveArtifacts 'target/*.jar'
                }
            }
        }
        stage('Code Quality Analysis') {
            parallel {
                stage('PMD') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn pmd:pmd'
                    }
                    post {
                        always {
                            recordIssues enabledForFailure: true, tool: pmdParser(pattern: '**/target/pmd.xml')
                        }
                    }
                }
                stage('Findbugs') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn findbugs:findbugs'
                    }
                    post {
                        always {
                            recordIssues enabledForFailure: true, tool: spotBugs(pattern: '**/target/findbugsXml.xml')
                        }
                    }
                }
           /*     stage('JavaDoc') {
                    agent {
                        docker {
                            image 'maven:3.6.0-jdk-8-alpine'
                            args '-v /root/.m2/repository:/root/.m2/repository'
                            reuseNode true
                        }
                    }
                    steps {
                        sh ' mvn javadoc:javadoc'
                        step([$class: 'JavadocArchiver', javadocDir: './target/site/apidocs', keepAll: 'true'])
                    }
                    post {
                        always {
                            recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc()]
                        }
                    }
                }*/
         /*   stage('SonarQube') {
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
                }*/
            }
        }
        stage('Deploy Artifact To Nexus') {
            when {
                anyOf { branch 'master'; branch 'develop' }
            }
            steps {
                script {
                    pom = readMavenPom file: 'pom.xml'
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
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
