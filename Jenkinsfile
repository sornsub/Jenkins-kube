def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any

    environment {
        registry = "sornsub/vproapp"
        registryCredential = "dockerhub"
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps { 
                withSonarQubeEnv('sonar-devsecops') { 
                    sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=devsecops -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml'   
                }
            }
        } 

        stage('Run SCA Analysis Snyk') {
            steps {     
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }       

        stage('BUILD APP DOCKER IMAGE') {
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('UPLOAD DOCKER IMAGE') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("REMOVE UNUSED DOCKER IMAGE") {
            steps {
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }
 
        stage("KUBERNETES DEPLOY") {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    
        stage('RunDASTUsingZAP') {
            agent { label 'KOPS' }
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {

                        sh "kubectl wait --namespace prod --for=condition=ready pod -l app=vproapp --timeout=300s"

                        def APP_HOST = ""

                        waitUntil {
                            sleep 5

                            APP_HOST = sh(
                                script: "kubectl get svc vproapp-service -n prod -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                                returnStdout: true
                            ).trim()

                            if (!APP_HOST) {
                                echo "Waiting for hostname..."
                                return false
                            }

                            def status = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' http://${APP_HOST}",
                                returnStdout: true
                            ).trim()

                            echo "Status: ${status}"

                            return status == "200"
                        }

                        echo "Target Ready: http://${APP_HOST}"
                    }

                    docker.image('softwaresecurityproject/zap-stable').inside('--entrypoint=""') {
                        sh "zap-baseline.py -t http://${APP_HOST} -r zap_report.html || true"
                    }
                    archiveArtifacts artifacts: 'zap_report.html'

                }
            }
        }



    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult] ?: '#cccccc',
                message: "*${currentBuild.currentResult} :* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}