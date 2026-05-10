pipeline {
    agent any

    tools {
        maven 'maven3.9.15'
        jdk 'java21'
    }

    environment {
        APP_NAME         = 'country-chicken-backend'

        // ✅ Nexus host/ports
        NEXUS_MAVEN_URL  = '3.26.3.72:8081'
        NEXUS_DOCKER_URL = '3.26.3.72:8081'

        MAVEN_REPO       = 'mvn-release'
        DOCKER_REPO      = 'docker-release'

        GROUP_ID         = 'com.countrychicken'
        VERSION          = ''
        JAR_NAME         = ''
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'git@github.com:yoganandadevops/countrychicken.git'
            }
        }

        stage('Set Version') {
            steps {
                script {
                    VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()

                    if (!VERSION) {
                        error "❌ Version not found from pom.xml"
                    }

                    JAR_NAME = "${APP_NAME}-${VERSION}.jar"
                    echo "✅ Version: ${VERSION}"
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests -s $HOME/.m2/settings.xml'
            }
        }

        stage('Upload JAR to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_MAVEN_URL}",
                    groupId: "${GROUP_ID}",
                    version: "${VERSION}",
                    repository: "${MAVEN_REPO}",
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [
                            artifactId: "${APP_NAME}",
                            classifier: '',
                            file: "target/${JAR_NAME}",
                            type: 'jar'
                        ]
                    ]
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build \
                  -t ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:${VERSION} \
                  -t ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:latest .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-nexus-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login ${NEXUS_DOCKER_URL} -u "$DOCKER_USER" --password-stdin

                    docker push ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:${VERSION}
                    docker push ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:latest

                    docker logout ${NEXUS_DOCKER_URL}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build & Push Successful"
        }
        failure {
            echo "❌ Build Failed"
        }
        always {
            sh 'docker system prune -f'
            cleanWs()
        }
    }
}

