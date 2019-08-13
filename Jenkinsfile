def readProperties()
{

	def properties_file_path = "${workspace}" + "@script/properties.yml"
	def property = readYaml file: properties_file_path
	env.APP_NAME = property.APP_NAME
    	env.MS_NAME = property.MS_NAME
    	env.BRANCH = property.BRANCH
    	env.GIT_SOURCE_URL = property.GIT_SOURCE_URL
	env.GIT_CREDENTIALS = property.GIT_CREDENTIALS
    	env.SONAR_HOST_URL = property.SONAR_HOST_URL
    	env.CODE_QUALITY = property.CODE_QUALITY
    	env.UNIT_TESTING = property.UNIT_TESTING
    	env.CODE_COVERAGE = property.CODE_COVERAGE
    	env.FUNCTIONAL_TESTING = property.FUNCTIONAL_TESTING
    	env.SECURITY_TESTING = property.SECURITY_TESTING
	env.PERFORMANCE_TESTING = property.PERFORMANCE_TESTING
	env.TESTING = property.TESTING
	env.QA = property.QA
	env.PT = property.PT
	env.User = property.User
	env.DOCKER_REGISTRY = property.DOCKER_REGISTRY
	env.mailrecipient = property.mailrecipient
	env.USER = property.USER
	env.REPO = property.REPO
	env.PR_TITLE = property.PR_TITLE
	env.PR_MSG = property.PR_MSG
	env.HEAD_BRANCH = property.HEAD_BRANCH
	env.BASE_BRANCH = property.BASE_BRANCH
	
	
    
}



def buildApp(projectName,msName){
openshift.withCluster() {
        openshift.withProject(projectName) {
            def bcSelector = openshift.selector( "bc", msName) 
            def bcExists = bcSelector.exists()
            if (!bcExists) {
                openshift.newBuild("https://github.com/Vageesha17/projsvc","--strategy=docker")
                sh 'sleep 400'               
            } else {
                sh 'echo build config already exists in development environment,starting existing build'  
                openshift.startBuild(msName,"--wait")                
            }
        }
}
}
def FAILED_STAGE
podTemplate(cloud:'openshift',label: 'selenium', 
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'cloudbees/jnlp-slave-with-java-build-tools',
      alwaysPullImage: true,
      args: '${computer.jnlpmac} ${computer.name}'
    ),
	 containerTemplate(
      name: 'docker',
      image: 'manya97/jenkins_tryout',
      alwaysPullImage: true,
      args: '${computer.jnlpmac} ${computer.name}',
      ttyEnabled: true
    )],volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),hostPathVolume(hostPath: '/etc/docker/daemon.json', mountPath: '/etc/docker/daemon.json')] )
{
node 
{
   def MAVEN_HOME = tool "MAVEN_HOME"
   def JAVA_HOME = tool "JAVA_HOME"
   env.PATH="${env.PATH}:${MAVEN_HOME}/bin:${JAVA_HOME}/bin"
	try{
   stage('Checkout')
   {
       FAILED_STAGE=env.STAGE_NAME
       readProperties()
       checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '${GIT_CREDENTIALS}', url: "${GIT_SOURCE_URL}"]]])
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:dev-apps --dry-run -o yaml >> Orchestration/deployment-dev.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:test-apps --dry-run -o yaml >> Orchestration/deployment-test.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:qa-apps --dry-run -o yaml >> Orchestration/deployment-qa.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:pt-apps --dry-run -o yaml >> Orchestration/deployment-pt.yaml'   
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:uat-apps --dry-run -o yaml >> Orchestration/deployment-uat.yaml'	   
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=${DOCKER_REGISTRY}/$APP_NAME-dev-apps/${MS_NAME}:preprod-apps --dry-run -o yaml >> Orchestration/deployment-preprod.yaml'
   }

   stage('Initial Setup')
   {
		FAILED_STAGE=env.STAGE_NAME
       		sh 'mvn clean compile'
   }
   if(env.UNIT_TESTING == 'True')
   {
        stage('Unit Testing')
        {
			FAILED_STAGE=env.STAGE_NAME
              sh 'mvn test'
        }
   }
  
   if(env.CODE_QUALITY == 'True')
   {
        stage('Code Quality Analysis')
        {
			  FAILED_STAGE=env.STAGE_NAME
              sh 'mvn sonar:sonar -Dsonar.host.url="${SONAR_HOST_URL}"'
        }
   }
    if(env.CODE_COVERAGE == 'True')
   {
      stage('Code Coverage')
   	{
		FAILED_STAGE=env.STAGE_NAME
        	sh 'mvn package -Djacoco.percentage.instruction=${EXPECTED_COVERAGE}'
       
   	}
   }
  if(env.SECURITY_TESTING == 'True')
  {
      stage('Security Testing')
      {
			FAILED_STAGE=env.STAGE_NAME
			sh 'mvn findbugs:findbugs'
      }	
  }
   
   

   stage('Dev - Build Application')
   {
	   FAILED_STAGE=env.STAGE_NAME
       buildApp("${APP_NAME}-dev-apps", "${MS_NAME}")
   }
   stage('Tagging Image for Dev')
   {
	   FAILED_STAGE=env.STAGE_NAME
	  node('selenium')
    {
        
        container('docker')
        {
            sh 'git clone https://github.com/abhinav-goyall/meteringservice.git'
            sh "docker build -t test '/home/jenkins/workspace/${env.JOB_NAME}/${MS_NAME}'"
            sh 'docker tag test ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:dev-apps'
            sh 'docker push ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:dev-apps'
        }
    }
	   
   }
   stage('Dev - Deploy Application')
   {
	   FAILED_STAGE=env.STAGE_NAME
       sh 'oc apply -f Orchestration/deployment-dev.yaml -n=${APP_NAME}-dev-apps'
       sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-dev-apps'
       
   }
    if(env.BRANCH != "master")	
    {		
	    stage('Pull Request Generation')
	   {

		withCredentials([usernamePassword(credentialsId: 'PullRequest_credentials', passwordVariable: 'password', usernameVariable: 'username')]) 
		{
		     sh """
		curl -k -X POST -u $username:$password https://api.github.com/repos/${USER}/${REPO}/pulls \
		-d  "{
		    \\"title\\": \\"${PR_TITLE}\\",
		    \\"body\\": \\"${PR_MSG}\\",
		    \\"head\\": \\"${USER}:${HEAD_BRANCH}\\",
		    \\"base\\": \\"${BASE_BRANCH}\\"
		}\""""
		    }
		    emailext body: "Pull Request has been raised. You can review at https://github.com/${USER}/${REPO}/pulls (Please open this in chrome) ", subject: "Pull Request Generated", to: '${mailrecipient}'
	    }	
    }	  	    
   if(env.BRANCH=="master")	
   {		
	   stage('Tagging Image for Testing')
	   { 
		   FAILED_STAGE=env.STAGE_NAME
		   node('selenium')
	    {

		container('docker')
		{
				sh 'docker pull ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:dev-apps'
		    sh 'docker tag ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:dev-apps ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:test-apps'
		    sh 'docker push ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:test-apps'
		}
	    }
	   }

	   if(env.TESTING == 'True')
	   {	
	       stage('Test - Deploy Application')
	       {
		       FAILED_STAGE=env.STAGE_NAME
		      sh 'oc apply -f Orchestration/deployment-test.yaml -n=${APP_NAME}-test-apps'
		      sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-test-apps'
	       }

	       node('selenium')
	       {

		  stage('Integration Testing')
		  {
			  FAILED_STAGE=env.STAGE_NAME
		      container('jnlp')
		      {

			   checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '${GIT_CREDENTIALS}', url: "${GIT_SOURCE_URL}"]]])
			   sh 'mvn integration-test'
		      }
		   }
		}
	    }

	if(env.QA == 'True')
	   {	
	       stage('Test - Deploy Application')
	       {
		       FAILED_STAGE=env.STAGE_NAME
		       sh 'oc apply -f Orchestration/deployment-qa.yaml -n=${APP_NAME}-qa-apps'
		       sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-qa-apps'
	       }
	       node('selenium')
	       {
		    stage('Integration Testing')
		    {
			    FAILED_STAGE=env.STAGE_NAME
			container('jnlp')
			{
			     checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '${GIT_CREDENTIALS}', url: "${GIT_SOURCE_URL}"]]])
			     sh 'mvn integration-test'
			}
		     }
		}
	    }
	stage('Tagging Image for PT')
	  {
		  FAILED_STAGE=env.STAGE_NAME
		node('selenium')
	    {

		container('docker')
		{
				sh 'docker pull ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:test-apps'
		    sh 'docker tag ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:test-apps ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:pt-apps'
		    sh 'docker push ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:pt-apps'
		}
	    } 
	   }
	if(env.PT == 'True')
	   {	

		stage('Test - Deploy Application')
		 {
			 FAILED_STAGE=env.STAGE_NAME
			sh 'oc apply -f Orchestration/deployment-pt.yaml -n=${APP_NAME}-pt-apps'
			sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-pt-apps'
		 }

		stage('Performance Testing')
		{
			FAILED_STAGE=env.STAGE_NAME
			sh 'mvn verify'
		}

	    }

		stage('Tagging Image for UAT')
		{
			FAILED_STAGE=env.STAGE_NAME
			node('selenium')
			{

				container('docker')
				{
					sh 'docker pull ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:pt-apps'
					sh 'docker tag ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:pt-apps ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:uat-apps'
					sh 'docker push ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:uat-apps'
				}
			} 
		}
		stage('Test - UAT Application')
		 {
			 FAILED_STAGE=env.STAGE_NAME
			sh 'oc apply -f Orchestration/deployment-uat.yaml -n=${APP_NAME}-uat-apps'
			sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-uat-apps'
		 }
		stage('Tagging Image for Pre-Prod')
		{
			FAILED_STAGE=env.STAGE_NAME
			node('selenium')
			{

				container('docker')
				{
					sh 'docker pull ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:uat-apps'
					sh 'docker tag ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:uat-apps ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:preprod-apps'
					sh 'docker push ${DOCKER_REGISTRY}/$APP_NAME-dev-apps/$MS_NAME:preprod-apps'
				}
			} 

		}
		stage('Test - Preprod Application')
		 {
			 FAILED_STAGE=env.STAGE_NAME
			sh 'oc apply -f Orchestration/deployment-preprod.yaml -n=${APP_NAME}-preprod-apps'
			sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-preprod-apps'
		 }
   	   }
	}
	catch(e){
		echo "Pipeline has failed"
		emailext body: "${env.BUILD_URL} has result ${currentBuild.result} at stage ${FAILED_STAGE} with error" + e.toString(), subject: "Failure of pipeline: ${currentBuild.fullDisplayName}", to: '${mailrecipent}'
		throw e
	}
	
 
}
}	
