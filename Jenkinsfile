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
    }
}