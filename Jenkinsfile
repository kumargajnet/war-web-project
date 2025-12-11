pipeline {
    agent any

    environment {
        // ---- Server & Credentials ----
        TOMCAT_SERVER = "13.201.119.9"
        TOMCAT_USER = "ec2-user"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"

        // ---- Nexus ----
        NEXUS_URL = "13.232.130.128:8081"              // ✅ No trailing slash
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "Nexus-credentials"

        // ---- SonarQube ----
        SONAR_HOST_URL = "http://13.235.70.174:9000"
        SONAR_CREDENTIAL_ID = "Jenkins_Sonar_token"    // ✅ Matches Jenkins credentials
    }

    tools {
        maven "maven"
    }

    stages {

        stage('Build WAR') {
            steps {
                echo "🔧 Building WAR package..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."
                withSonarQubeEnv('SonarQube Server') {
                    withCredentials([string(credentialsId: "${SONAR_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=wwp \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN} \
                              -Dsonar.java.binaries=target/classes
                        """
                    }
                }
            }
        }

        // stage('Quality Gate') {
        //     steps {
        //         echo "🧱 Waiting for SonarQube quality gate result..."
        //         timeout(time: 10, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('Extract Version') {
            steps {
                script {
                    echo "📦 Extracting version from pom.xml..."
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "Project Version: ${env.ART_VERSION}"
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    echo "⬆️ Uploading WAR to Nexus repository..."
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    echo "Uploading file: ${warFile}"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        groupId: 'koddas.web.war',
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [[
                            artifactId: 'wwp',
                            classifier: '',
                            file: warFile,
                            type: 'war'
                        ]]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    echo "🚀 Deploying WAR to Tomcat server..."
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()

                    sh """
                        scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                        ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} '
                            sudo mv /tmp/*.war /opt/tomcat/webapps/ &&
                            sudo systemctl restart tomcat
                        '
                    """
                }
            }
        }

        stage('Display URLs') {
            steps {
                script {
                    def appUrl = "http://${TOMCAT_SERVER}:8080/wwp-${ART_VERSION}"
                    def nexusUrl = "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${ART_VERSION}/wwp-${ART_VERSION}.war"
                    
                    echo "🌐 Application URL: ${appUrl}"
                    echo "📦 Nexus Artifact URL: ${nexusUrl}"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs for errors.'
        }
    }
}
