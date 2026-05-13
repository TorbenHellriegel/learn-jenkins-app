pipeline {
    agent any

    stages {
        // Comment Build Stage
        /*stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }*/
            /*
            do steps
            */
            /*steps {
                sh '''
                    #this will be ignored in shell
                    echo "Cleaning old workspace:"
                    rm -rf node_modules build

                    echo "Build Stage"
                    echo "Current file directory:"
                    ls -la

                    echo "Used versions for build:"
                    node --version
                    npm --version

                    echo "Executing build:"
                    npm ci
                    npm run build

                    echo "Final file directory:"
                    ls -la
                '''
            }
        }*/
        stage('Test') {
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
        }
        stage('E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.60.0-noble'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test
                '''
            }
        }
    }

    post{
        always{
            junit 'test-results/junit.xml'
        }
    }
}
