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

                archiveArtifacts artifacts: 'target/*.jar',
                                 fingerprint: true

                archiveArtifacts artifacts: 'target/build-info.txt',
                                 fingerprint: true
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
                    echo ========================================
                    echo Verifying Docker availability
                    echo ========================================
                    docker --version

                    echo.
                    echo ========================================
                    echo Building staging Docker image
                    echo ========================================
                    docker build -t sit753-petclinic:staging .

                    echo.
                    echo ========================================
                    echo Removing previous staging container
                    echo ========================================
                    docker rm -f sit753-petclinic-staging 2>NUL || echo No previous staging container found.

                    echo.
                    echo ========================================
                    echo Starting new staging container
                    echo ========================================
                    docker run -d ^
                    --name sit753-petclinic-staging ^
                    -p 8081:8080 ^
                    sit753-petclinic:staging

                    echo.
                    echo ========================================
                    echo Current running containers
                    echo ========================================
                    docker ps
                '''

                echo 'Waiting for Spring PetClinic staging environment to initialise...'

                sleep time: 10, unit: 'SECONDS'

                bat '''
                    echo ========================================
                    echo Checking staging application health
                    echo ========================================

                    for /L %%i in (1,1,12) do (

                        echo.
                        echo Health check attempt %%i of 12...

                        curl --fail --silent --show-error ^
                        http://localhost:8081/actuator/health ^
                        > staging-health.txt 2>NUL

                        if not errorlevel 1 (
                            echo.
                            echo Staging health response:
                            type staging-health.txt
                            echo.
                            echo.
                            echo ========================================
                            echo Staging deployment health check PASSED.
                            echo ========================================
                            exit /b 0
                        )

                        echo Application is not ready yet.
                        echo Waiting 5 seconds before retrying...

                        powershell -NoProfile -Command "Start-Sleep -Seconds 5"
                    )

                    echo.
                    echo ========================================
                    echo ERROR: Staging application did not become healthy.
                    echo ========================================

                    echo.
                    echo Staging container logs:
                    docker logs sit753-petclinic-staging

                    exit /b 1
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'staging-health.txt',
                                     allowEmptyArchive: true
                }
            }
        }


        stage('Release') {
            steps {
                echo 'Promoting tested staging image to production...'

                bat '''
                    echo ========================================
                    echo Creating versioned release image
                    echo ========================================

                    docker tag sit753-petclinic:staging sit753-petclinic:v1.0.%BUILD_NUMBER%

                    echo.
                    echo ========================================
                    echo Creating production tag
                    echo ========================================

                    docker tag sit753-petclinic:staging sit753-petclinic:production

                    echo.
                    echo ========================================
                    echo Removing previous production container
                    echo ========================================

                    docker rm -f sit753-petclinic-production 2>NUL || echo No previous production container found.

                    echo.
                    echo ========================================
                    echo Starting production container
                    echo ========================================

                    docker run -d ^
                    --name sit753-petclinic-production ^
                    -p 8082:8080 ^
                    sit753-petclinic:v1.0.%BUILD_NUMBER%

                    echo.
                    echo ========================================
                    echo Production container
                    echo ========================================

                    docker ps --filter "name=sit753-petclinic-production"
                '''

                echo 'Waiting for production application to initialise...'

                sleep time: 10, unit: 'SECONDS'

                bat '''
                    echo ========================================
                    echo Verifying production health
                    echo ========================================

                    for /L %%i in (1,1,12) do (

                        echo.
                        echo Production health check attempt %%i of 12...

                        curl --fail --silent --show-error ^
                        http://localhost:8082/actuator/health ^
                        > production-health.txt 2>NUL

                        if not errorlevel 1 (
                            echo.
                            echo Production health response:
                            type production-health.txt
                            echo.
                            echo.
                            echo ========================================
                            echo Production release health check PASSED.
                            echo ========================================
                            exit /b 0
                        )

                        echo Production application is not ready yet.
                        echo Waiting 5 seconds before retrying...

                        powershell -NoProfile -Command "Start-Sleep -Seconds 5"
                    )

                    echo.
                    echo ========================================
                    echo ERROR: Production application did not become healthy.
                    echo ========================================

                    echo.
                    echo Production container logs:
                    docker logs sit753-petclinic-production

                    exit /b 1
                '''

                bat '''
                    echo Release Version: v1.0.%BUILD_NUMBER% > release-info.txt
                    echo Jenkins Build: %BUILD_NUMBER% >> release-info.txt
                    echo Environment: Production >> release-info.txt
                    echo Production URL: http://localhost:8082 >> release-info.txt
                    echo Staging URL: http://localhost:8081 >> release-info.txt
                '''

                archiveArtifacts artifacts: 'release-info.txt',
                                 allowEmptyArchive: false

                archiveArtifacts artifacts: 'production-health.txt',
                                 allowEmptyArchive: true
            }
        }
    }
}