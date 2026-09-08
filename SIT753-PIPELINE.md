# SIT753 7.3HD DevOps Pipeline

## Project

Spring PetClinic has been selected as the application for the SIT753 7.3HD DevOps pipeline task.

Spring PetClinic is an existing open-source Spring Boot web application. The application contains backend logic, database operations, automated tests and multiple functional components.

## DevOps Pipeline

The Jenkins pipeline developed for this task will contain the following seven stages:

1. Build
2. Test
3. Code Quality
4. Security
5. Deploy
6. Release
7. Monitoring

## Planned Technologies

- Application: Spring PetClinic
- Language: Java
- Framework: Spring Boot
- Build Tool: Maven
- CI/CD: Jenkins
- Testing: JUnit
- Code Quality: SonarQube
- Security: Trivy
- Deployment: Docker
- Monitoring: Prometheus and Grafana

## Build Automation

The Jenkins pipeline uses SCM polling to automatically detect changes pushed to the GitHub repository.