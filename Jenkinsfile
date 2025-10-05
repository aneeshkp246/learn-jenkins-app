pipeline {
    agent any

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

                stage('E2E') {
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
                                reportName: 'Playwright HTML Report', 
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
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Install netlify-cli and use node directly to bypass user checks
                    export npm_config_cache=${WORKSPACE}/.npm-cache
                    npm install netlify-cli node-fetch@2
                    
                    # Use netlify via npx which handles the user issue better
                    npx netlify --version
                    npx netlify status
                '''
            }
        }
    }
}