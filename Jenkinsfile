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
                    dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
                }
            }
        }

        stage('UPLOAD DOCKER IMAGE') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("REMOVE UNUSED DOCKER IMAGE") {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER} || true"
            }
        }

        stage("KUBERNETES DEPLOY") {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }

        stage('GET APP URL FROM KOPS') {
            agent { label 'KOPS' }
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        sh "kubectl wait --namespace prod --for=condition=ready pod -l app=vproapp --timeout=300s"

                        waitUntil {
                            sleep 5

                            def host = sh(
                                script: "kubectl get svc vproapp-service -n prod -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                                returnStdout: true
                            ).trim()

                            if (!host) {
                                echo "Waiting for hostname..."
                                return false
                            }

                            def status = sh(
                                script: """
                                    code=\$(curl -s -o /dev/null -w '%{http_code}' --connect-timeout 5 http://${host} || true)
                                    [ -n "\$code" ] && echo "\$code" || echo "000"
                                """,
                                returnStdout: true
                            ).trim()

                            echo "Status from KOPS: ${status}"

                            if (status == "200" || status == "301" || status == "302") {
                                writeFile file: 'app_url.txt', text: "http://${host}"
                                return true
                            }

                            return false
                        }
                    }

                    stash name: 'app-url', includes: 'app_url.txt'
                }
            }
        }

        stage('RunDASTUsingZAP') {
            agent any
            steps {
                script {
                    unstash 'app-url'

                    def APP_URL = readFile('app_url.txt').trim()

                    if (!APP_URL) {
                        error("APP_URL is empty. Cannot run ZAP scan.")
                    }

                    echo "Running ZAP scan against ${APP_URL}"

                    docker.image('softwaresecurityproject/zap-stable').inside('--entrypoint=""') {
                        sh "zap-baseline.py -t ${APP_URL} -r zap_report.html || true"
                    }

                    archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
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