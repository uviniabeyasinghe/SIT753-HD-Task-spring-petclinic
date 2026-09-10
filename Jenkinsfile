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

        stage('Release') {
            steps {
                echo 'Promoting tested staging image to production...'

                bat '''
                    echo Creating versioned release image...
                    docker tag sit753-petclinic:staging sit753-petclinic:v1.0.%BUILD_NUMBER%

                    echo Creating production tag...
                    docker tag sit753-petclinic:staging sit753-petclinic:production

                    echo Removing previous production container if it exists...
                    docker rm -f sit753-petclinic-production 2>NUL || echo No previous production container found.

                    echo Starting production container...
                    docker run -d ^
                    --name sit753-petclinic-production ^
                    -p 8082:8080 ^
                    sit753-petclinic:v1.0.%BUILD_NUMBER%

                    echo Production container:
                    docker ps --filter "name=sit753-petclinic-production"
                '''

                echo 'Waiting for production application to start...'

                sleep time: 15, unit: 'SECONDS'

                bat '''
                    echo Verifying production health...
                    curl --fail http://localhost:8082/actuator/health

                    if %ERRORLEVEL% NEQ 0 (
                        echo ERROR: Production health check failed.
                        docker logs sit753-petclinic-production
                        exit /b 1
                    )

                    echo Production release health check PASSED.

                    echo Release Version: v1.0.%BUILD_NUMBER% > release-info.txt
                    echo Jenkins Build: %BUILD_NUMBER% >> release-info.txt
                    echo Environment: Production >> release-info.txt
                    echo Production URL: http://localhost:8082 >> release-info.txt
                '''

                archiveArtifacts artifacts: 'release-info.txt',
                                allowEmptyArchive: false
            }
        }
    }
}