pipeline {
    agent any
    stages {
        stage ('Build Backend') {
            steps {
                sh'''
                  ./mvnw clean package -DskipTests=true
                '''
            }
        }
        stage ('Unit Tests') {
            steps {
                sh'''
                  cd ${WORKSPACE}
                  ./mvnw test
                '''
            }
        }
        stage ('Sonar Analysis') {
            #environment {
            #    scannerHome = tool 'sonarScanner'
            #}
	    def scannerHome = tool 'sonarScanner';
            steps {
                withSonarQubeEnv('SONAR_LOCAL') {
                    sh'''
                     ${scannerHome}/bin/sonar-scanner -e -Dsonar.projectKey=Backend -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=08c561cdf322910bd8ad94f9d41ecd9c8aa1e6a5 -Dsonar.java.binaries=target -Dsonar.coverage.exclusions=**/../mvnw/**,**/src/test/**,**/model/**,**Application.java
                    '''
                }
            }
        }
        stage ('Quality Gate') {
            steps {
                sleep(5)
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage ('Deploy Backend') {
            steps {
                deploy adapters: [tomcat8(credentialsId: 'tomcat-login', path: '', url: 'http://tomcat:8000/')], contextPath: 'tasks-backend', war: 'target/tasks-backend.war'
            }
        }
        stage ('API Test') {
            steps {
                dir('api-test') {
                    git credentialsId: 'github-secret', url: 'https://github.com/wcaquino/tasks-api-test'
                    sh'''
                      cd ${WORKSPACE}
                      ./mvnw test
                    '''
                }
            }
        }
        stage ('Deploy Frontend') {
            steps {
                dir('frontend') {
                    git credentialsId: 'github-secret', url: 'https://github.com/wcaquino/tasks-frontend'
                    sh'''
                      cd ${WORKSPACE}
                      ./mvnw clean package
                      deploy adapters: [tomcat8(credentialsId: 'tomcat-login', path: '', url: 'http://tomcat:8000/')], contextPath: 'tasks', war: 'target/tasks.war'
                    '''
                }
            }
        }
        stage ('Functional Test') {
            steps {
                dir('functional-test') {
                    git credentialsId: 'github-secret', url: 'https://github.com/wcaquino/tasks-functional-tests'
                    sh'''
                      cd ${WORKSPACE}
                      ./mvnw test
                    '''
                }
            }
        }
        stage('Deploy Prod') {
            steps {
                sh'''
                  docker-compose build
                  docker-compose up -d
                '''
            }
        }
        stage ('Health Check') {
            steps {
                sleep(5)
                dir('functional-test') {
                    sh'''
                      ./mvnw verify -Dskip.surefire.tests
                    '''
                }
            }
        }
    }
    post {
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml, api-test/target/surefire-reports/*.xml, functional-test/target/surefire-reports/*.xml, functional-test/target/failsafe-reports/*.xml'
            archiveArtifacts artifacts: 'target/tasks-backend.war, frontend/target/tasks.war', onlyIfSuccessful: true
        }
        unsuccessful {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build $BUILD_NUMBER has failed', to: 'wcaquino+jenkins@gmail.com'
        }
        fixed {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build is fine!!!', to: 'wcaquino+jenkins@gmail.com'
        }
    }
}


