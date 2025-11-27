pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'prod'], description: 'Target environment')
    }

    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    environment {
        DOCKER_IMAGE = "rihemakkari/spring-petclinic:${BUILD_NUMBER}"
        DOCKER_REGISTRY = "docker.io"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rihemakkari/spring-petclinic.git',
                    credentialsId: 'github-creds-id'
            }
        }

        stage('Build') {
            steps {
                sh 'rm -rf .scannerwork'
                sh './mvnw clean compile'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key	', variable: 'b4d8b5ef-5bed-4ef2-8bbf-4d47d470954e')]) {
                    dependencyCheck(
                        odcInstallation: 'owasp-dependency',
                        additionalArguments: '--suppression suppression.xml --enableRetired --format HTML --format XML --scan . --nvdApiKey ${NVD_API_KEY}',
                        stopBuild: false,
                        skipOnScmChange: false
                    )
                }
            }
            post {
                always {
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    archiveArtifacts artifacts: 'dependency-check-report.*', allowEmptyArchive: true
                }
            }
        }

        stage('Secrets Scan (Gitleaks)') {
            steps {
                sh 'gitleaks detect -s . --report-format=json --report-path=gitleaks-report.json'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test -Dtest=!PostgresIntegrationTests'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('sq1') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName='Spring PetClinic' \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.tests=src/test/java \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.java.test.binaries=target/test-classes \
                            -Dsonar.junit.reportPaths=target/surefire-reports \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                            -Dsonar.java.source=25
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                sh 'rm -rf .scannerwork'
            }
        }

        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker logout
                    '''
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Full CI/CD pipeline completed!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
