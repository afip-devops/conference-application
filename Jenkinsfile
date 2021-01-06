pipeline {
    agent any

    tools {
        maven "Maven 3.6.3"
    }
    options {
        parallelsAlwaysFailFast()
    }
        stages {
            stage('Conf-App Build Maven') {
                steps {
                    git url: 'https://github.com/aptlt/conference-application.git'
                    sh "mvn clean install package"
                    archiveArtifacts artifacts: '**/*.war', followSymlinks: false
                }
            }
            stage('Conf-App Analyse Sonar') {
                steps {
                    withSonarQubeEnv(installationName: 'SonarQube', credentialsId: 'sonarqube'){
                        sh "mvn clean package sonar:sonar -Dsonar.host_url=$SONAR_HOST_URL"
                    }
                }
            }
            stage('Conf-App Publication Nexus') {
                steps {
                    nexusPublisher nexusInstanceId: 'Nexus', nexusRepositoryId: 'maven-releases', packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: '', filePath: 'target/conference-app-3.0.0.war']], mavenCoordinate: [artifactId: 'conference-app', groupId: 'de.codecentric', packaging: 'war', version: '3.0.0']]]
                }
            }
            stage('Conf-App Docker Build Image') {
                steps {
                    sh "rm -Rf conference-app-3.0.0.war && \
                    wget http://172.81.183.213:18081/repository/maven-releases/de/codecentric/conference-app/3.0.0/conference-app-3.0.0.war -O ${WORKSPACE}/conference-app-3.0.0.war && \
                    docker build -t conference-app:latest ."
                }
            }
            stage('Conf-App Push Image on Docker Hub') {
                steps {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-aptlt', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                        sh "docker login -u $USERNAME -p $PASSWORD && \
                        docker image tag conference-app $USERNAME/conference-app-3.0.0 && \
                        docker push $USERNAME/conference-app-3.0.0"
                    }
                }
            }
            stage('Conf-App Install Docker on Remote Server') {
                steps {
                    ansibleTower jobTemplate: 'Install docker conference-app', jobType: 'run', throwExceptionWhenFail: false, towerCredentialsId: 'awx', towerLogLevel: 'full', towerServer: 'AWX'
                }
            }
            stage('Conf-App Run Application') {
                steps {
                    ansibleTower jobTemplate: 'Run application conference-app', jobType: 'run', throwExceptionWhenFail: false, towerCredentialsId: 'awx', towerLogLevel: 'full', towerServer: 'AWX'
                }
            }
            stage('Parallel Stage') {
                parallel {
                    stage('Conf-App JMeter') {
                        steps {
                            sh "jmeter -Jjmeter.save.saveservice.output_format=xml -Jjmeter.save.saveservice.response_data.on_error=true -n -t test_plan_conference_app.jmx -l testresult.jlt"
                        }
                    }
                    stage('Conf-App Selenium') {
                        steps {
                            git 'https://github.com/aptlt/conference-application.git'
                            sh "mvn test"
                        }
                    }
                }
            }
        }
}
