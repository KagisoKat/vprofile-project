pipeline {
    agent any
    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin1995'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vprofile-maven-central'
        NEXUSIP = '172.31.19.252'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vprofile-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
    }

    stages {
        stage('Debug Environment') {
            steps {
                echo "NEXUS_GRP_REPO: ${NEXUS_GRP_REPO}"
                echo "NEXUSIP: ${NEXUSIP}"
                echo "NEXUSPORT: ${NEXUSPORT}"
                sh 'mvn --version'
                sh 'java -version'
            }
        }

        stage('Build') {
            steps {
                script {
                    try {
                        echo 'Attempting build with Nexus repository...'
                        sh 'mvn -s settings.xml -DskipTests install'
                    } catch (Exception e) {
                        echo 'Build failed with Nexus settings, trying with fallback settings...'
                        try {
                            sh 'mvn -s settings-fallback.xml -DskipTests install'
                        } catch (Exception e2) {
                            echo 'Build failed with fallback settings, trying with default Maven Central...'
                            sh 'mvn -DskipTests install'
                        }
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    try {
                        echo 'Running unit tests...'
                        sh 'mvn -s settings.xml test'
                    } catch (Exception e) {
                        echo 'Tests failed with Nexus settings, trying with fallback...'
                        try {
                            sh 'mvn -s settings-fallback.xml test'
                        } catch (Exception e2) {
                            sh 'mvn test'
                        }
                    }
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Code Quality Analysis') {
            steps {
                script {
                    try {
                        echo 'Running code quality analysis...'
                        sh 'mvn -s settings.xml sonar:sonar'
                    } catch (Exception e) {
                        echo 'SonarQube analysis failed, continuing with build...'
                    }
                }
            }
        }

        stage('Package & Deploy to Nexus') {
            steps {
                script {
                    def version = readMavenPom().getVersion()
                    def artifactId = readMavenPom().getArtifactId()
                    def groupId = readMavenPom().getGroupId()

                    echo "Deploying ${groupId}:${artifactId}:${version} to Nexus..."

                    if (version.endsWith('-SNAPSHOT')) {
                        echo 'Deploying SNAPSHOT version to snapshot repository'
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                            groupId: "${groupId}",
                            version: "${version}",
                            repository: "${SNAP_REPO}",
                            credentialsId: "${NEXUS_LOGIN}",
                            artifacts: [
                                [artifactId: "${artifactId}",
                                 classifier: '',
                                 file: "target/${artifactId}-${version}.war",
                                 type: 'war']
                            ]
                        )
                    } else {
                        echo 'Deploying RELEASE version to release repository'
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                            groupId: "${groupId}",
                            version: "${version}",
                            repository: "${RELEASE_REPO}",
                            credentialsId: "${NEXUS_LOGIN}",
                            artifacts: [
                                [artifactId: "${artifactId}",
                                 classifier: '',
                                 file: "target/${artifactId}-${version}.war",
                                 type: 'war']
                            ]
                        )
                    }
                }
            }
        }
    }
}
