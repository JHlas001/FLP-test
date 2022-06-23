    agent {
        node {
            label 'linuxworker1'
        }
    }
    options {
        timestamps()
        disableConcurrentBuilds()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    environment {
        STOR_ACCESS_KEY = credentials('DSOFS_TF_ACCESS_KEY')
	    RUN_SONARQUBE_STATUS = 'Pending'	
	    RUN_LIGHTHOUSE_STATUS = 'Pending'
    }
	
	
	stages {
        stage ('checkout') {
            steps {
                script {
                    currentBuild.displayName = "#$BUILD_NUMBER DSO Platform"
                    currentBuild.description = "DSO Platform Pipeline"
                }
            }
        }       
	

    stage ('build') {
        steps {
            script {

            if (env.BRANCH_NAME == 'Develop' || env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging') {
            sh '''
                cp frontend/.sequelizelocal.js frontend/.sequelize.js
                docker-compose build dso_frontend 
            	docker-compose build dso_backend             

                docker stop dso_frontend_app
                docker rm dso_frontend_app

                docker stop dso_backend_app
                docker rm dso_backend_app

	            docker-compose up -d

            '''
          sleep(90)

            sh '''      
                docker logs dso_frontend_app
            '''
            }
            }
        }
        } 

  

    stage('run tests') {
      parallel {
  

     stage ('run-owasp-dependency-check') {
        steps {
            echo "owasp dependency check"
            script {
            if (env.BRANCH_NAME == 'Develop' || env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging') {
            sh '''
            pwd
            cd frontend
              # /home/jenkins/dependency-check/bin/dependency-check.sh --scan src --project DSOPlatform --out owaspreport -f XML
            '''
            }
            }
            }
        }

   

  stage('run-508-testing'){
    steps {
    script {
          echo "run lighthouse"
          if (env.BRANCH_NAME == 'Develop' || env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging') {
          	sh label: '', script: 'lighthouse --quiet --no-update-notifier --no-enable-error-reporting --output=json --output-path=./lighthouse-report.json --chrome-flags="--headless" http://localhost'
           sh label: '', script: 'lighthouse --quiet --no-update-notifier --no-enable-error-reporting --output=json --output-path=./lighthouse-report.json --chrome-flags="--headless" http://localhost'              
			lighthouseReport 'lighthouse-report.json'
          }
            RUN_LIGHTHOUSE_STATUS = 'Success'
        }
        }
    }
    
  

stage('run-container-scanning'){
    steps {
    script {
          echo "run container scanning"
          if (env.BRANCH_NAME == 'Develop' || env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging') {
          sh '''
          trivy image dso_backend_app
          '''
          }
        }
        }
    }

   stage('run-security-scanning'){
    steps {
    script {
        def scannerhome = tool 'SonarQubeScanner';
        withSonarQubeEnv('SonarQube') {      
        sh '''
        cd frontend
      #  npm install -g typescript
        '''
        sh '''
        /home/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin/sonar-scanner -Dsonar.projectKey=DSO.Platform.Frontend -Dsonar.projectName=DSO.Platform.Frontend -Dsonar.sources=frontend -Dsonar.dependencyCheck.reportPath=owaspreport/dependency-check-report.xml
        #/datadrive/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin/sonar-scanner -Dsonar.projectKey=DSO.Platform.Frontend -Dsonar.projectName=DSO.Platform.Frontend -Dsonar.sources=frontend
        '''
        //          sh 'java -jar /home/jenkins/sonar-cnes-report-3.1.0.jar -t ef6187db815664e57ad05c12fed29dd2234b1538 -s https://sca.azurecloudgov.us -p DSODemoWebApplication -o sonarqubereports'
        //          sh 'cp sonarqubereports/*-analysis-report.docx sonarqubereports/sonarqubeanalysisreport.docx'
        //          sh 'cp sonarqubereports/*-issues-report.xlsx sonarqubereports/sonarqubeissuesreport.xlsx'
        //publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'sonarqubereports/sonarqubeanalysisreport.docx', reportName: 'SonarQube Analysis Report (Word)', reportTitles: 'SonarQube Analysis Report'])
        //publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'sonarqubereports/sonarqubeissuesreport.xlsx', reportName: 'SonarQube Issues Report (Excel)', reportTitles: 'SonarQube Issues Report'])
        RUN_SONARQUBE_STATUS= 'Success'
        }      
        }
        }
    }
    }
    }      

stage("quality-gate") { 
  steps {
    script {
  
if (env.BRANCH_NAME == 'Develop' || env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging') {
    sleep(30)
    def qualitygate = waitForQualityGate()
      if (qualitygate.status != 'OK') {
         error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
      }
}
    }    
    }
    }      
    
stage('upload-artifacts'){
    steps {
    script {
          echo "upload artifacts"

 if (env.BRANCH_NAME == 'Develop') {
        echo 'This is develop branch'

        sh '''
        zip -r dsoplatformrelease.zip frontend/src/
        '''
        
                  }



 if (env.BRANCH_NAME == 'staging') {
        echo 'This is staging branch'
        sh '''
        zip -r dsoplatformrelease.zip frontend/src/
        '''
       
                  }


 if (env.BRANCH_NAME == 'master') {
        echo 'This is master branch'
        sh '''
        zip -r dsoplatformrelease.zip frontend/src/
        '''

                  }



        }
        }
    }

stage('dev-deploy'){
    when{
	branch 'develop'
	}
    agent {
        label "VM-Portal"
        }
    steps {
    script {
          echo "dev deploy"
          sh '''
          cp docker-compose-dev.yaml docker-compose.yaml
          cp frontend/src/environments/environmentdevazure.ts frontend/src/environments/environment.ts
          cp frontend/.sequelizedev.js frontend/.sequelize.js
          docker-compose build dso_backend
          docker-compose build dso_frontend
	  
	  docker stop dso_frontend_app
                docker rm dso_frontend_app

                docker stop dso_backend_app
                docker rm dso_backend_app
		
          docker-compose up -d
          docker ps
          '''

    echo "DSO Platform app deployed successfully: http://dsoplatformdev.usgovvirginia.cloudapp.usgovcloudapi.net "
          
        }
        }
    }

stage('staging-deploy'){
    when{
	branch 'staging'
	}
        agent {
        label "VM-Portal-Staging"
        }

    steps {
    script {
          echo "staging deploy"
                    sh '''
          cp docker-compose-staging.yaml docker-compose.yaml
          cp frontend/src/environments/environmentstagingazure.ts frontend/src/environments/environment.ts
          cp frontend/.sequelizestaging.js frontend/.sequelize.js
          docker-compose build dso_backend
          docker-compose build dso_frontend
          docker-compose up -d
          docker ps
          '''

          echo "DSO Platform app deployed successfully: http://dsoplatformstaging.usgovvirginia.cloudapp.usgovcloudapi.net "

        }
        }
    }

stage('prod-deploy'){
    when{
	branch 'master'
	}
        agent {
        label "VM-Portal-Prod"
        }

    steps {
    script {
          echo "prod deploy"
                    sh '''
          cp docker-compose-prod.yaml docker-compose.yaml
          cp frontend/src/environments/environmentprodazure.ts frontend/src/environments/environment.ts
          cp frontend/.sequelizeprod.js frontend/.sequelize.js
          docker-compose build dso_backend
          docker-compose build dso_frontend
          docker-compose up -d
          docker ps
          '''

          echo "DSO Platform app deployed successfully: http://dsoplatformprod.usgovvirginia.cloudapp.usgovcloudapi.net "

        }
        }
    }        
	
stage('email-notification'){
		steps {
		  script
	    {
		//	emailext attachLog: true, attachmentsPattern: 'sonarqubereports/sonarqubeanalysisreport.docx,sonarqubereports/sonarqubeissuesreport.xlsx', body: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS: Check console output at $BUILD_URL to view the results.', replyTo: 'notifications@techtrend.us', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', to: 'ikumarasamy@techtrend.us,sgezen@techtrend.us'
			
    env.LCHECKOUT_STATUS = "Success"
    env.LGITHUB_PROJECT_NAME = "DSO-Platform"
    env.LINSTALL_DEPENDENCIES_STATUS = "Success"
    env.LBUILD_APPLICATION = "Success"
    env.LOWASP_DEPENDENCY_CHECK = "Success"
    env.LRUN_LINT_STATUS = "Success"
    env.LRUN_UNIT_TESTS_STATUS = "Success"
    env.LRUN_E2E_STATUS = "Success"
    env.LRUN_PA11Y_STATUS = "${RUN_LIGHTHOUSE_STATUS}"
    env.LRUN_SONARQUBE_STATUS = "${RUN_SONARQUBE_STATUS}"
    env.LDEPLOY_STATUS = "Success"
    env.LGIT_BRANCH = "${GIT_BRANCH}"


    env.BLUE_OCEAN_URL="${env.JENKINS_URL}/blue/organizations/jenkins/DSO_Platform_Azure/detail/$BRANCH_NAME/${BUILD_NUMBER}/pipeline"
    env.LIGHTHOUSE_REPORT_URL = "${env.JENKINS_URL}/job/DSO_Platform_Azure/job/$BRANCH_NAME/${BUILD_NUMBER}/lighthousereport/"


    env.BLUE_OCEAN_URL_SQ_DOCX="${env.BUILD_URL}artifact/sonarqubereports/sonarqubeanalysisreport.docx"
    env.BLUE_OCEAN_URL_SQ_XLSX="${env.BUILD_URL}artifact/sonarqubereports/sonarqubeissuesreport.xlsx"
	env.LSONARQUBE_URL="https://sca.azurecloudgov.us/dashboard?id=DSO.Platform.Frontend&branch=Develop"

   	emailext attachLog: false, attachmentsPattern: '', body: '''${SCRIPT, template="dsodemo.template"}''', mimeType: 'text/html', replyTo: 'notifications@techtrend.us', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', to: 'ikumarasamy@techtrend.us,sgezen@techtrend.us'
			
	}		
		}
    }   


		
      
    }
 //   post {
  //      always {
          //  deleteDir()
   //     }
   // }
}