pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
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
        }
    }
}
