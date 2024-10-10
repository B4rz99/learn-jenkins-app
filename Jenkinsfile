pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1bb5a64b-8891-4df8-842f-f89a0b5eaa5d'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            environment {
                AWS_S3_BUCKET = 'obarbozaa-learn-jenkins'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'AWS', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        echo "Deploying to AWS S3" > index.html
                        aws s3 cp index.html s3://$AWS_S3_BUCKET/index.html
                    '''
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'playwright'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                    docker {
                        image 'playwright'
                        reuseNode true
                    }
                }
            
            environment {
                CI_ENVIRONMENT_URL = 'TBS'
            }

                steps {
                    sh '''
                        netlify --version
                        echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                        netlify status
                        netlify deploy --dir=build --json > deploy-output.json
                        CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                        npx playwright test  --reporter=html
                    '''
                }
                post {
                    always {
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod Report', reportTitles: '', useWrapperFileDirectly: true])
                    }
                }
            }

        stage('Deploy production') {
            agent {
                    docker {
                        image 'playwright'
                        reuseNode true
                    }
                }

            environment {
                CI_ENVIRONMENT_URL = 'https://vocal-caramel-53300e.netlify.app'
            }

                steps {
                    sh '''
                        node --version
                        netlify --version
                        echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                        netlify status
                        netlify deploy --dir=build --prod
                        npx playwright test  --reporter=html
                    '''
                }
                post {
                    always {
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod Report', reportTitles: '', useWrapperFileDirectly: true])
                    }
                }
            } 
    }
}
