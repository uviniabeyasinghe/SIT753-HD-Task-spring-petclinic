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


        stage('Monitoring') {
            steps {
                echo 'Verifying production monitoring and alerting services...'

                /*
                 * First confirm the three main services themselves
                 * are reachable before asking Prometheus for the
                 * production target state.
                 */
                bat '''
                    echo ========================================
                    echo Monitoring Verification
                    echo ========================================

                    echo.
                    echo [1/4] Checking production application...

                    curl --fail --silent --show-error ^
                    http://localhost:8082/actuator/health ^
                    > monitoring-production-health.txt

                    if errorlevel 1 (
                        echo ERROR: Production application is not healthy.
                        exit /b 1
                    )

                    type monitoring-production-health.txt

                    echo.
                    echo Production application health check PASSED.


                    echo.
                    echo [2/4] Checking Prometheus readiness...

                    curl --fail --silent --show-error ^
                    http://localhost:9090/-/ready ^
                    > monitoring-prometheus-ready.txt

                    if errorlevel 1 (
                        echo ERROR: Prometheus is not ready.
                        exit /b 1
                    )

                    type monitoring-prometheus-ready.txt

                    echo.
                    echo Prometheus readiness check PASSED.


                    echo.
                    echo [3/4] Checking Alertmanager readiness...

                    curl --fail --silent --show-error ^
                    http://localhost:9093/-/ready ^
                    > monitoring-alertmanager-ready.txt

                    if errorlevel 1 (
                        echo ERROR: Alertmanager is not ready.
                        exit /b 1
                    )

                    type monitoring-alertmanager-ready.txt

                    echo.
                    echo Alertmanager readiness check PASSED.


                    echo.
                    echo [4/4] Checking Prometheus production target...
                '''


                /*
                 * Prometheus may temporarily report 0 immediately
                 * after Release recreates the production container.
                 *
                 * Retry instead of failing on the first scrape.
                 */
                powershell '''
                    Write-Host ""
                    Write-Host "========================================"
                    Write-Host "Prometheus Production Target Verification"
                    Write-Host "========================================"
                    Write-Host ""

                    Write-Host "Waiting for Prometheus to detect the new production deployment..."

                    $maxAttempts = 12
                    $targetHealthy = $false

                    for ($i = 1; $i -le $maxAttempts; $i++) {

                        Write-Host ""
                        Write-Host "Prometheus target check attempt $i of $maxAttempts..."

                        try {

                            $response = Invoke-RestMethod `
                                -Uri 'http://localhost:9090/api/v1/query?query=up%7Bjob%3D%22spring-petclinic-production%22%7D'

                            $response |
                                ConvertTo-Json -Depth 10 |
                                Out-File -Encoding utf8 monitoring-prometheus-target.json

                            if ($response.status -eq 'success') {

                                if ($response.data.result.Count -gt 0) {

                                    $targetValue = [string]$response.data.result[0].value[1]

                                    Write-Host "Production target value: $targetValue"

                                    if ($targetValue -eq '1') {

                                        Write-Host ""
                                        Write-Host "Prometheus production target is UP."

                                        $targetHealthy = $true
                                        break
                                    }
                                    else {
                                        Write-Host "Prometheus currently reports the production target as DOWN."
                                    }
                                }
                                else {
                                    Write-Host "Production target is not available in the Prometheus query result yet."
                                }
                            }
                            else {
                                Write-Host "Prometheus query did not return success."
                            }
                        }
                        catch {
                            Write-Host "Prometheus query is not ready yet."
                            Write-Host $_.Exception.Message
                        }

                        if ($i -lt $maxAttempts) {
                            Write-Host "Waiting 5 seconds before retrying..."
                            Start-Sleep -Seconds 5
                        }
                    }

                    if (-not $targetHealthy) {

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "ERROR: Monitoring verification failed"
                        Write-Host "========================================"

                        Write-Error "Production monitoring target did not become UP after $maxAttempts attempts."
                        exit 1
                    }

                    Write-Host ""
                    Write-Host "========================================"
                    Write-Host "Prometheus production target check PASSED"
                    Write-Host "========================================"
                '''


                /*
                 * Create a summary artifact after all monitoring
                 * checks have passed.
                 */
                bat '''
                    echo.
                    echo ========================================
                    echo Monitoring Verification PASSED
                    echo ========================================

                    echo Application: Spring PetClinic > monitoring-status.txt
                    echo Environment: Production >> monitoring-status.txt
                    echo Production URL: http://localhost:8082 >> monitoring-status.txt
                    echo Prometheus URL: http://localhost:9090 >> monitoring-status.txt
                    echo Alertmanager URL: http://localhost:9093 >> monitoring-status.txt
                    echo Monitoring Target: UP >> monitoring-status.txt
                    echo Jenkins Build: %BUILD_NUMBER% >> monitoring-status.txt
                    echo Version: %APP_VERSION% >> monitoring-status.txt

                    echo.
                    echo ===== Monitoring Summary =====
                    type monitoring-status.txt
                    echo ==============================
                '''

                archiveArtifacts artifacts: 'monitoring-*.txt, monitoring-*.json',
                                 allowEmptyArchive: false
            }
        }
    }
}