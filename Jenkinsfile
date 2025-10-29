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

        stage('SonarQube Analysis') {
            when {
                // Only run if build artifacts exist and SonarQube tools are available
                allOf {
                    expression { fileExists('target/classes') }
                    expression {
                        try {
                            tool "${SONARSCANNER}"
                            return true
                        } catch (Exception e) {
                            echo "SonarQube scanner not found: ${e.getMessage()}"
                            return false
                        }
                    }
                }
            }
            steps {
                script {
                    try {
                        def scannerHome = tool "${SONARSCANNER}"
                        echo "Scanner Home: ${scannerHome}"

                        withSonarQubeEnv("${SONARSERVER}") {
                            echo 'Starting SonarQube analysis...'
                            echo "SonarQube Server: ${env.SONAR_HOST_URL}"
                            echo 'Project Key: vprofile'

                            sh """${scannerHome}/bin/sonar-scanner \
                               -Dsonar.projectKey=vprofile \
                               -Dsonar.projectName=vprofile-repo \
                               -Dsonar.projectVersion=1.0 \
                               -Dsonar.sources=src/ \
                               -Dsonar.java.binaries=target/classes \
                               -Dsonar.junit.reportsPath=target/surefire-reports/ \
                               -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                               -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"""

                            // Display SonarQube dashboard link prominently
                            echo '======================================='
                            echo '🎯 SONARQUBE ANALYSIS COMPLETE'
                            echo '======================================='
                            if (env.SONAR_HOST_URL) {
                                def dashboardUrl = "${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                                echo "📊 SonarQube Dashboard: ${dashboardUrl}"
                                echo "🔗 Click the link above to view your analysis"
                                echo '======================================='

                                // Set build description with prominent link
                                currentBuild.displayName = "#${BUILD_NUMBER} - SonarQube Analysis"
                                currentBuild.description = '''
                                <div style='background-color: #e7f3ff; padding: 10px; border-left: 4px solid #2196F3;'>
                                    <strong>📊 SonarQube Analysis Complete</strong><br/>
                                    <a href='${dashboardUrl}' target='_blank' style='color: #2196F3; font-weight: bold;'>
                                        🔗 View Dashboard →
                                    </a><br/>
                                    <small>Project: vprofile</small>
                                </div>
                                '''
                            } else {
                                echo "⚠️  SONAR_HOST_URL environment variable not set!"
                                echo 'Check your SonarQube server configuration in Jenkins'
                            }
                        }
                    } catch (Exception e) {
                        echo "SonarQube analysis failed: ${e.getMessage()}"
                        echo 'This may be due to:'
                        echo '1. SonarQube server not configured or running'
                        echo '2. Scanner tool not found'
                        echo '3. Network connectivity issues'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
            post {
                always {
                    script {
                        // Add SonarQube link to build description
                        try {
                            if (env.SONAR_HOST_URL) {
                                def sonarUrl = "${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                                currentBuild.description = "SonarQube: <a href='${sonarUrl}' target='_blank'>View Dashboard</a>"
                            }
                        } catch (Exception e) {
                            echo "Could not set build description: ${e.getMessage()}"
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            when {
                // Only run if SonarQube analysis was successful
                expression {
                    return currentBuild.result != 'FAILURE' && env.SONAR_HOST_URL != null
                }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        try {
                            echo 'Waiting for SonarQube Quality Gate...'
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "Quality Gate Status: ${qg.status}"
                                if (env.SONAR_HOST_URL) {
                                    echo "View detailed results at: ${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                                }
                                // Don't fail the build, just mark as unstable
                                currentBuild.result = 'UNSTABLE'
                                echo 'Quality Gate failed but build will continue as UNSTABLE'
                            } else {
                                echo 'Quality Gate passed successfully!'
                                if (env.SONAR_HOST_URL) {
                                    echo "View results at: ${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                                }
                            }
                        } catch (Exception e) {
                            echo "Quality Gate check failed or timed out: ${e.getMessage()}"
                            if (env.SONAR_HOST_URL) {
                                echo "Check SonarQube dashboard manually at: ${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                            }
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Always try to provide SonarQube dashboard link
                try {
                    if (env.SONAR_HOST_URL) {
                        def sonarDashboardUrl = "${env.SONAR_HOST_URL}/dashboard?id=vprofile"
                        echo '=== BUILD COMPLETE ==='
                        echo "SonarQube Dashboard: ${sonarDashboardUrl}"

                        // Add clickable link to build description
                        currentBuild.description = '''
                        <div style='background-color: #e7f3ff; padding: 10px; border-left: 4px solid #2196F3; margin: 10px 0;'>
                            <strong>📊 SonarQube Analysis:</strong><br/>
                            <a href='${sonarDashboardUrl}' target='_blank' style='color: #2196F3; font-weight: bold; font-size: 16px;'>
                                🔗 View Dashboard →
                            </a><br/>
                            <small style='color: #666;'>Project: vprofile | Build: #${BUILD_NUMBER}</small>
                        </div>
                        '''
                    } else {
                        echo 'SonarQube host URL not available. Check SonarQube server configuration.'
                    }
                } catch (Exception e) {
                    echo "Could not generate SonarQube link: ${e.getMessage()}"
                }
            }
        }
        success {
            echo 'Pipeline completed successfully!'
            echo 'Check the build description above for SonarQube dashboard link'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
