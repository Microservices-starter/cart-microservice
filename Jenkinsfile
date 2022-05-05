def getVersion(){
    def commitHash =  sh returnStdout: true, script: 'git rev-parse --short HEAD'
    return commitHash
}

pipeline{
    agent any

    options{
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }

    environment{
        GIT_COMMIT_HASH = getVersion()
        cart = "${cart}"
        scannerHome = "${JENKINS_HOME}"

    }

    stages{
        stage("Code checkout"){
            steps{
                echo "[INFO] Checking out latest code from git"
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/Microservices-starter/cart-microservice.git'  
            }
        }

        stage("Sonar analysis"){
            steps{
                echo "[INFO] Performing analysis with Sonarqube"
                script{
                    withSonarQubeEnv(credentialsId: 'sonartoken'){
                        sh "sudo ${scannerHome}/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=$cart"
                    }
                }
            }

        }

        stage("Quality Gates"){
            steps{
                echo "[INFO] Verifying Quality gates"
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Docker build"){
            steps{
                echo "[INFO] Building Docker images"
                sh 'docker build -t rajputmarch2020/cart:$GIT_COMMIT_HASH .'
            }
        }

        stage("Trivy Scan"){
            steps{
                echo "[INFO] Scanning image with Trivy"
                sh 'trivy image rajputmarch2020/cart:$GIT_COMMIT_HASH'
            }
        }

        stage("Docker push"){
            steps{
                echo "[INFO] Pushing Docker images to Dockerhub"
                withCredentials([string(credentialsId: 'dockerhub', variable: 'password')]){
                    sh 'docker login -u rajputmarch2020 -p ${password} '
                }
                    sh 'docker push rajputmarch2020/cart:$GIT_COMMIT_HASH'
            }
        }

        stage("Teardown"){
            steps{
                echo "[INFO] Deleting Docker images after pushed to Dockerhub"
                sh ''' 
                  docker rmi rajputmarch2020/cart:$GIT_COMMIT_HASH
                  docker image prune -f
                '''
            }
        }

        stage("Helm charts Config check"){
            steps{
                echo "[INFO] Checking Helm chart config"
                script{
                    dir('helm-charts'){
                        withEnv(['DATREE_TOKEN=ao1RpL3G3LMRL6eucy37hv']){
                            sh 'helm datree test cart/'
                        }
                    }
                }
            }
        }

        stage("Approval"){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "ravisinghrajput005@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }
    }

    post{
        always{
            echo "========always========"
        }
        success{
            echo "========pipeline executed successfully ========"
        }
        failure{
            echo "========pipeline execution failed========"
        }
        changed{
            echo "========build has changed from last run========"
        }
    }
}