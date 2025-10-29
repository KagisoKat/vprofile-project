pipeline {
    agent any
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
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
        
        stage('Build'){
            steps {
                script {
                    try {
                        echo "Attempting build with Nexus repository..."
                        sh 'mvn -s settings.xml -DskipTests install'
                    } catch (Exception e) {
                        echo "Build failed with Nexus settings, trying with fallback settings..."
                        try {
                            sh 'mvn -s settings-fallback.xml -DskipTests install'
                        } catch (Exception e2) {
                            echo "Build failed with fallback settings, trying with default Maven Central..."
                            sh 'mvn -DskipTests install'
                        }
                    }
                }
            }
        }
    }
}