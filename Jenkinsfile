pipeline {
    agent any
    
    environment {
        NETLIFY_SITE_ID = '46bdd434-0bca-4499-af30-292d9f866a04'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {
        stage('Docker') {
            steps {
                sh 'docker build -t my_playwright .'
            }
        }

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
                sh '''
                    # this will be ignored in shell
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
                    # npm install serve netlify-cli@20.1.1 node-jq

                    echo "Executing build:"
                    npm run build

                    echo "Final file directory:"
                    ls -la
                '''
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
                            image 'my_playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Test', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my_playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to Staging, Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > staging-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node-jq -r '.deploy_url' staging-output.json", returnStdout: true)
                }
            }
        }

        stage('Staging E2E Test') {
            agent {
                docker {
                    image 'my_playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "$env.STAGING_URL"
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Stage', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        /*stage('Approval') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input 'Ready to Deploy?'
                }
            }
        }*/

        stage('Deploy Prod and E2E Test') {
            agent {
                docker {
                    image 'my_playwright'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://dashing-puppy-46fb12.netlify.app'
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to Production, Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Prod', reportTitles: '', useWrapperFileDirectly: true])
                }
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
