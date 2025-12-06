pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

    environment {
        MONGO_URI          = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_Creds     = credentials('mongo-db-credentials')
        MONGO_USERNAME     = credentials('mongo-db-username')
        MONGO_PASSWORD     = credentials('mongo-db-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610'
    }

    stages {

        stage('VM Node Version') {
            steps {
                sh '''
                    node -v
                    npm -v
                '''
            }
        }

        stage('Installing Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('Dependency Scanning') {
            parallel {

                stage('NPM Dependency Audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical
                            echo $?
                        '''
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan ./
                            --out ./
                            --format 'ALL'
                            --prettyPrint
                        ''', odcInstallation: 'OWASP-DepCheck-10'

                        dependencyCheckPublisher(
                            failedTotalCritical: 1,
                            pattern: 'dependency-check-report.xml',
                            stopBuild: true
                        )

                        junit allowEmptyResults: true,
                              keepProperties: true,
                              testResults: 'dependency-check-junit.xml'

                        publishHTML([
                            allowMissing: true,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: './',
                            reportFiles: 'dependency-check-jenkins.html',
                            reportName: 'Dependency Check HTML Report',
                            reportTitles: '',
                            useWrapperFileDirectly: true
                        ])
                    }
                }
            }
        }

        stage('Unit Testing') {
            options { retry(2) }
            steps {
                sh 'npm test'
                junit allowEmptyResults: true,
                      stdoutRetention: '',
                      testResults: 'test-results.xml'
            }
        }

        stage('Code Coverage') {
            steps {
                catchError(
                    buildResult: 'SUCCESS',
                    message: 'Oops! It will be fixed in future releases',
                    stageResult: 'UNSTABLE'
                ) {
                    sh 'npm run coverage'
                }

                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'coverage/lcov-report',
                    reportFiles: 'index.html',
                    reportName: 'Code Coverage HTML Report'
                ])
            }
        }

        stage('SAST - SonarQube') {
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    withSonarQubeEnv('sonar-qube-server') {
                        sh 'echo $SONAR_SCANNER_HOME'
                        sh '''
                            $SONAR_SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectKey=Solar-System-Project \
                            -Dsonar.sources=app.js \
                            -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info
                        '''
                    }
                }
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t siddharth67/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-hub-credentials', url: "") {
                    sh 'docker push siddharth67/solar-system:$GIT_COMMIT'
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  stdoutRetention: '',
                  testResults: 'test-results.xml'

            junit allowEmptyResults: true,
                  stdoutRetention: '',
                  testResults: 'dependency-check-junit.xml'

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/lcov-report',
                reportFiles: 'index.html',
                reportName: 'Code Coverage HTML Report'
            ])
        }
    }
}
