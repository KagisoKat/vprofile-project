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
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Debug Environment') {
            steps {
                echo '=== Environment Debug Information ==='
                echo "NEXUS_GRP_REPO: ${NEXUS_GRP_REPO}"
                echo "NEXUSIP: ${NEXUSIP}"
                echo "NEXUSPORT: ${NEXUSPORT}"
                echo "Workspace: ${WORKSPACE}"
                sh 'mvn --version'
                sh 'java -version'
                sh 'ls -la'
                sh 'pwd'
            }
        }

        stage('Build') {
            steps {
                script {
                    try {
                        echo 'Attempting build with Nexus repository...'
                        sh 'mvn -s settings.xml -DskipTests clean install'
                    } catch (Exception e) {
                        echo "Build failed with Nexus settings: ${e.getMessage()}"
                        echo 'Trying with fallback settings...'
                        try {
                            sh 'mvn -s settings-fallback.xml -DskipTests clean install'
                        } catch (Exception e2) {
                            echo "Build failed with fallback settings: ${e2.getMessage()}"
                            echo 'Trying with default Maven Central...'
                            sh 'mvn -DskipTests clean install'
                        }
                    }
                }
            }
            post {
                success {
                    echo 'Build completed successfully'
                    archiveArtifacts artifacts: 'target/*.war', allowEmptyArchive: true
                }
                failure {
                    echo 'Build failed'
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
                            echo 'Tests failed with fallback, trying with default...'
                            sh 'mvn test'
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists('target/surefire-reports/*.xml')) {
                            junit allowEmptyResults: true, testResultsPattern: 'target/surefire-reports/*.xml'
                        } else {
                            echo 'No test results found'
                        }
                    }
                }
            }
        }

        // Temporarily disabled - enable after basic build is working
        /*
        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    script {
                        try {
                            echo "Starting SonarQube analysis..."
                            sh '''${scannerHome}/bin/sonar-scanner \
                               -Dsonar.projectKey=vprofile \
                               -Dsonar.projectName=vprofile-repo \
                               -Dsonar.projectVersion=1.0 \
                               -Dsonar.sources=src/ \
                               -Dsonar.java.binaries=target/classes \
                               -Dsonar.junit.reportsPath=target/surefire-reports/ \
                               -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                               -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                        } catch (Exception e) {
                            echo "SonarQube analysis failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    script {
                        try {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            } else {
                                echo "Quality Gate passed successfully!"
                            }
                        } catch (Exception e) {
                            echo "Quality Gate check failed or timed out: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        */
    }
}
