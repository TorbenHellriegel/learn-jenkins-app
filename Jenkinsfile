pipeline {
    agent any
    
    environment {
        NETLIFY_SITE_ID = '46bdd434-0bca-4499-af30-292d9f866a04'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        // Comment Build Stage
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            /*
            do steps
            */
            steps {
                /*sh '''
                    #this will be ignored in shell
                    echo "Cleaning old workspace:"
                    rm -rf node_modules build

                    echo "Build Stage"
                    echo "Current file directory:"
                    ls -la

                    echo "Used versions for build:"
                    node --version
                    npm --version

                    echo "Installing:"
                    npm ci
                    npm install serve
                    npm install netlify-cli@20.1.1

                    echo "Executing build:"
                    npm run build

                    echo "Final file directory:"
                    ls -la
                '''*/
                echo 'Skipping Build Step...'
            }
        }
        stage('Run Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "Test Stage"
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post{
                        always{
                            junit 'jtest-results/junit.xml'
                        }
                    }
                }
                stage('E2E Test') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
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
                    node_modules/.bin/netlify --version
                    echo "Deploying to Production, Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
    /*post{
        always{
            junit 'jtest-results/junit.xml'
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }*/
}
