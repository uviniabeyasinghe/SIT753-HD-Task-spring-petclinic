pipeline {
    agent any

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        stage('Pipeline Setup') {
            steps {
                echo 'SIT753 7.3HD Jenkins DevOps Pipeline'
                echo 'Spring PetClinic source code successfully loaded from GitHub.'

                bat 'java -version'
                bat 'git --version'

                bat '''
                    if exist pom.xml (
                        echo Maven project detected successfully.
                    ) else (
                        echo ERROR: pom.xml was not found.
                        exit /b 1
                    )
                '''
            }
        }

        stage('Build') {
            steps {

                script {
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} - v${env.APP_VERSION}"
                }

                echo "Building Spring PetClinic version ${env.APP_VERSION}"

                bat 'call mvnw.cmd -B --no-transfer-progress -DskipTests clean package'

                bat '''
                    echo Application: Spring PetClinic > target\\build-info.txt
                    echo Version: %APP_VERSION% >> target\\build-info.txt
                    echo Jenkins Build Number: %BUILD_NUMBER% >> target\\build-info.txt
                    echo Git Commit: %GIT_COMMIT% >> target\\build-info.txt

                    echo.
                    echo ===== Generated JAR Artefact =====
                    dir target\\*.jar
                    echo ==================================
                '''

                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                archiveArtifacts artifacts: 'target/build-info.txt', fingerprint: true
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated Spring PetClinic tests...'

                bat 'call mvnw.cmd -B --no-transfer-progress test jacoco:report'
            }

            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: false

                    archiveArtifacts artifacts: 'target/site/jacoco/**',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Code Quality') {
            steps {
                echo 'Running SonarQube code quality analysis...'

                withSonarQubeEnv('SonarQube') {
                    bat '''
                        call mvnw.cmd -B --no-transfer-progress ^
                        org.sonarsource.scanner.maven:sonar-maven-plugin:sonar ^
                        -Dsonar.projectKey=sit753-petclinic ^
                        -Dsonar.projectName="Spring PetClinic - SIT753" ^
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Security') {
            steps {
                echo 'Running Trivy security scan...'

                bat '''
                    if not exist security-reports mkdir security-reports

                    "C:\\Trivy\\trivy.exe" fs ^
                    --scanners vuln,secret,misconfig ^
                    --severity HIGH,CRITICAL ^
                    --format table ^
                    --output security-reports\\trivy-high-critical-report.txt .
                '''

                script {
                    def trivyStatus = bat(
                        returnStatus: true,
                        script: '''
                            "C:\\Trivy\\trivy.exe" fs ^
                            --scanners vuln ^
                            --severity HIGH,CRITICAL ^
                            --exit-code 1 .
                        '''
                    )

                    if (trivyStatus != 0) {
                        error 'Security Gate FAILED: HIGH or CRITICAL dependency vulnerabilities were detected.'
                    }
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: 'security-reports/*.txt',
                                    allowEmptyArchive: true
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying Spring PetClinic to Docker staging environment...'

                bat '''
                    echo Verifying Docker availability...
                    docker --version

                    echo Building staging Docker image...
                    docker build -t sit753-petclinic:staging .

                    echo Removing previous staging container if it exists...
                    docker rm -f sit753-petclinic-staging 2>NUL || echo No previous staging container found.

                    echo Starting new staging container...
                    docker run -d ^
                    --name sit753-petclinic-staging ^
                    -p 8081:8080 ^
                    sit753-petclinic:staging

                    echo Current running containers:
                    docker ps
                '''

                echo 'Waiting for Spring PetClinic to start...'

                sleep time: 15, unit: 'SECONDS'

                bat '''
                    echo Checking application health...
                    curl --fail http://localhost:8081/actuator/health

                    if %ERRORLEVEL% NEQ 0 (
                        echo ERROR: Staging application health check failed.
                        docker logs sit753-petclinic-staging
                        exit /b 1
                    )

                    echo Staging deployment health check passed.
                '''
            }
        }
    }
}