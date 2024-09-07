pipeline {

    agent any
/*
	tools {
        maven "maven3"
    }
*/
    environment {
        registry = "imranvisualpath/vproappdock"
        registryCredentials = 'dockerhub'
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages{

        
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'pipeline {
    agent any

    environment {
        // Define your environment variables here
        scannerHome = tool name: 'sonarscanner4'
        registry = 'my-registry-url'
        registryCredentials = 'my-registry-credentials'
        NEXUS_VERSION = '3'
        NEXUS_PROTOCOL = 'https'
        NEXUS_URL = 'http://nexus.example.com'
        NEXUS_REPOSITORY = 'my-repo'
        NEXUS_CREDENTIAL_ID = 'my-nexus-credentials'
        ARTVERSION = '1.0'
    }

    stages {
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/ \
                       -Dsonar.java.binaries=target/classes/ \
                       -Dsonar.junit.reportsPath=target/surefire-reports/ \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified()}"
                    def artifactPath = filesByGlob[0].path
                    def artifactExists = fileExists artifactPath
                    if (artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION"
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: artifactPath,
                                 type: pom.packaging],
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: "pom.xml",
                                 type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    def dockerImage = "${registry}:$BUILD_NUMBER"
                    docker.build(dockerImage)
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredentials) {
                        docker.image("${registry}:$BUILD_NUMBER").push("V$BUILD_NUMBER")
                        docker.image("${registry}:$BUILD_NUMBER").push('latest')
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                script {
                    sh "docker rmi ${registry}:V$BUILD_NUMBER"
                }
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
