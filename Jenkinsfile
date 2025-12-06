pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }
    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_Creds = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
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
                catchError(buildResult: 'SUCCESS', message: 'Oops! It will be fixed in future releases', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report'])
            }
        }
    }
    post {
        always {
            junit allowEmptyResults: true, stdoutRetention: '', testResults: 'test-results.xml'
            junit allowEmptyResults: true, stdoutRetention: '', testResults: 'dependency-check-junit.xml'
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report'])
        }
    }
}
