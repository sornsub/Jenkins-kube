def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {

    agent any
/*
	tools {
        maven "maven3"
    }
*/
    environment {
        registry = "sornsub/vproapp"
        registryCredential = "dockerhub"
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages{

        stage('BUILD'){
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

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
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

/* <---
        stage('CODE ANALYSIS WITH SONARQUBE'){

            environment {
                scannerHome = tool 'sonar4.7'
            }

            steps {
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
---> */
        stage('BUILD APP DOCKER IMAGE'){
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('UPLOAD DOCKER IMAGE'){
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
            steps{
                sh 'docker rmi $registry:V$BUILD_NUMBER'
            }
        }
 
        stage("KUBERNETES DEPLOY") {
            agent {label 'KOPS'}
                steps{
                    sh 'helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod'
                }
        }
    
	    stage('RunDASTUsingZAP') {
            agent {label 'KOPS'}
                steps {
                    sh '''
                        APP_HOST=$(kubectl get svc vproapp-service -n prod -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                        echo "APP_HOST=$APP_HOST"

                        if [ -z "$APP_HOST" ]; then
                        echo "LoadBalancer hostname not found"
                        exit 1
                        fi

                        zap.sh -cmd -quickurl http://$APP_HOST -quickprogress -quickout ${WORKSPACE}/zap_report.html
                    '''
			     	archiveArtifacts artifacts: 'zap_report.html'
		        }	        
        } 

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild. currentResult],
                message: "*${currentBuild.currentResult} :* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }

    }
}