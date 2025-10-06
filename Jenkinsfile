pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '5f7d1e4f-88a0-4e44-8156-6cc7e9b509f1'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Emergency Cleanup') {
            steps {
                sh 'sudo rm -rf jest-results playwright-report || true'
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
                    # Set npm cache to workspace to avoid permission issues
                    export npm_config_cache=${WORKSPACE}/.npm-cache
                    
                    echo "Building with Docker"
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        echo 'Test stage'
                        sh '''
                            export npm_config_cache=${WORKSPACE}/.npm-cache
                            echo "Running tests with Docker"
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit '**/junit*.xml'
                        }
                    }
                }

                stage('Local E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        echo 'E2E stage'
                        sh '''
                            export npm_config_cache=${WORKSPACE}/.npm-cache
                            echo "Running E2E tests with Docker"
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false, 
                                alwaysLinkToLastBuild: false, 
                                keepAll: false, 
                                reportDir: 'playwright-report', 
                                reportFiles: 'index.html', 
                                reportName: 'Playwright HTML Report for Local', 
                                reportTitles: '', 
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            agent {
                dockerfile {
                    filename 'Dockerfile.deploy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    export npm_config_cache=${WORKSPACE}/.npm-cache
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    node_modules/.bin/netlify status
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify deploy --dir=build --prod --no-build --site $NETLIFY_SITE_ID
                '''
            }
        }
        stage('Prod E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    environment {
                        CI_ENVIRONMENT_URL = 'https://guileless-lokum-460112.netlify.app'
                    }
                    steps {
                        echo 'E2E stage'
                        sh '''
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false, 
                                alwaysLinkToLastBuild: false, 
                                keepAll: false, 
                                reportDir: 'playwright-report', 
                                reportFiles: 'index.html', 
                                reportName: 'Playwright HTML Report for Production', 
                                reportTitles: '', 
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
    }
}