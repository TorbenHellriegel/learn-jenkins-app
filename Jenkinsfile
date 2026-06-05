pipeline {
    agent any
    
    environment {
        NETLIFY_SITE_ID = '46bdd434-0bca-4499-af30-292d9f866a04' //this variable name must be set because netlify directly looks for this value
        NETLIFY_AUTH_TOKEN = credentials('netlify-token') //the credentials are set in the jenkins web-interface and can be accessed with credentials()
        REACT_APP_VERSION = "1.0.$BUILD_ID" //this is used in the App.js to automatically increment the app version with each deployment (the $BUILD_ID comes from Jenkins)
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine' //alpine is a smaller compact docker container for npm
                    reuseNode true //this is important so that the workspace can be reused by other stages. otherwise a file created here could not be accessed by other stages
                }
            }
            /*
            in the sh we:
            - clean up the old workspace
            - clean-install (ci) the neccessary packages
            - build the application
            */
            steps {
                sh '''
                    # this comment will be ignored in shell
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
                //echo 'Skipping Build Step...'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myJenkinsApp .'
            }
        }

        stage('Run Tests') {
            parallel { //this causes the steps defined within to be run in parrallel to save time. can only be done if there are no dependencies between them
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        //in this sh we do a simple test if the index.html was created correctly and then rund the tests defined in the code with npm
                        sh '''
                            echo "Test Stage"
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post{ //is run after all steps are done
                        always{ //is always run, can also define other options for success, failure, etc.
                            junit 'jtest-results/junit.xml' //this provides the testresults to e displayed directly in the jenkins web-inerface
                        }
                    }
                }
                stage('E2E Test') {
                    agent {
                        docker {
                            image 'my_playwright' //this docker image is previously created in a seperate jekins pipeline from a dockerfile. see below for more info
                            reuseNode true
                        }
                    }
                    steps {
                        //in this sh we start a server to enable us to make e2e tests then sleep for 10 s to give the server enough time to start and finally execute the playwright test
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Test', reportTitles: '', useWrapperFileDirectly: true]) //after the playwright test we provide the results in jekins with this command
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my_playwright' //the my_playwright image was custom made because we reuse it multiple times
                    reuseNode true
                }
            }
            steps {
                //in this sh we deploy the app to netlify but not to production. we also save the json output to a file to later acces and test the staging environment
                sh '''
                    netlify --version
                    echo "Deploying to Staging, Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > staging-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node-jq -r '.deploy_url' staging-output.json", returnStdout: true) //this saves the staging url from the json output in an env to use in a later stage
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
                CI_ENVIRONMENT_URL = "$env.STAGING_URL" //get the staging url from env. the CI_ENVIRONMENT_URL is another fixed variable name that netlify checks for directly
            }
            steps {
                //this sh runs the playwrite test again this time in the staging environment instead of the local build like before
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Stage', reportTitles: '', useWrapperFileDirectly: true]) //again we show the test results in jenkins
                }
            }
        }

        /*stage('Approval') { //the approval stage is opional and can be used to manually decide if we want to proceed with deploying to production
            steps {
                timeout(time: 5, unit: 'MINUTES') { 
                    input 'Ready to Deploy?' //this function waits 5 min for an ok and aborts the dwployment if no input is given
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
                CI_ENVIRONMENT_URL = 'https://dashing-puppy-46fb12.netlify.app' //unlike the staging here we give the final productin website where we want to deploy
            }
            steps {
                //in this sh we deploy to prod (with the --prod tag) and test the deployment right after. this is done in one step to shorten the number of steps in jenkins for better overview. they can also be split into deploy and test like before with the staging deployment
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
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report Prod', reportTitles: '', useWrapperFileDirectly: true]) //again we show the test results in jenkins
                }
            }
        }

        stage('Deploy AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli' //to access aws we need a docker image with the aws-cli. not giving a specific versin will default this to latest
                    reuseNode true
                    args "-u root --entrypoint=''" //the entrypoint needs to be set for the aws-cli to work properly and the root user is needen to install jq later
                }
            }
            environment {
                AWS_S3_BUCKET = 'learn-jenkins-20260601' //the name of the bucket in aws (needs to be created beforehand with an aws account)
                AWS_DEFAULT_REGION = 'us-east-1' //this is needed when using task definitions to set the region of the cluster
                AWS_ECS_CLUSTER = 'LearnJenkinsApp-Cluster-Prod' //self defines variables
                AWS_ECS_SERVICE = 'LearnJenkinsApp-Service-Prod' //self defines variables
                AWS_ECS_TASK = 'LearnJenkinsApp-TaskDefinition-Prod' //self defines variables
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) { //the credentials are saved as username and password in the jenkins web-interface beforehand. the withCredentials command comes from the aws-cli
                    //in the sh we create an example file and send it to the bucket in aws
                    /*sh '''
                        aws --version
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''*/
                    //in this sh we deploy our app with clusters and task definitions instead (see task definition below). the LearnJenkinsApp Cluster and Service names are set in the aws web interface. jq is needed to handle variables between the commands. at the end we wait for the service to be fully deployed
                    sh '''
                        aws --version
                        yum install jq -y
                        LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revision')
                        echo $LATEST_TD_REVISION
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE --task-definition $AWS_ECS_TASK:$LATEST_TD_REVISION
                        aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICE
                    '''
                }
            }
        }
    }

    //this post "stage" is always executed after all the stages are done. it is no longer used here but remeins as a commt for future reference
    /*post{
        always{
            junit 'jtest-results/junit.xml'
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }*/
}
