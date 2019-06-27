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
	env.mailrecipent = property.mailrecipent
	
    
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

podTemplate(cloud:'openshift',label: 'selenium', 
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'cloudbees/jnlp-slave-with-java-build-tools',
      alwaysPullImage: true,
      args: '${computer.jnlpmac} ${computer.name}'
    )])
{
node 
{
   def MAVEN_HOME = tool "MAVEN_HOME"
   def JAVA_HOME = tool "JAVA_HOME"
   env.PATH="${env.PATH}:${MAVEN_HOME}/bin:${JAVA_HOME}/bin"
	try{
   stage('Checkout')
   {
       readProperties()
       checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '${GIT_CREDENTIALS}', url: "${GIT_SOURCE_URL}"]]])
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:dev-apps --dry-run -o yaml >> Orchestration/deployment-dev.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:test-apps --dry-run -o yaml >> Orchestration/deployment-test.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:qa-apps --dry-run -o yaml >> Orchestration/deployment-qa.yaml'
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:pt-apps --dry-run -o yaml >> Orchestration/deployment-pt.yaml'   
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:uat-apps --dry-run -o yaml >> Orchestration/deployment-uat.yaml'	   
       sh 'oc set image --local=true -f Orchestration/deployment.yaml ${MS_NAME}=13.89.217.239:5000/$APP_NAME-dev-apps/${MS_NAME}:preprod-apps --dry-run -o yaml >> Orchestration/deployment-preprod.yaml'
       sh 'docker login -u $User -p "$(oc whoami -t)" docker-registry-default.40.71.221.144.nip.io' 
   }

   stage('Initial Setup')
   {
       sh 'mvn clean compile'
   }
   if(env.UNIT_TESTING == 'True')
   {
        stage('Unit Testing')
        {
              sh 'mvn test'
        }
   }
   if(env.CODE_COVERAGE == 'True')
   {
        stage('Code Coverage')
        {
        sh 'mvn package'
        }
   }
   if(env.CODE_QUALITY == 'True')
   {
        stage('Code Quality Analysis')
        {
              sh 'mvn sonar:sonar -Dsonar.host.url="${SONAR_HOST_URL}"'
        }
   }
  if(env.SECURITY_TESTING == 'True')
  {
      stage('Security Testing')
      {
        sh 'mvn findbugs:findbugs'
      }	
  }
   
   

   stage('Dev - Build Application')
   {
       buildApp("${APP_NAME}-dev-apps", "${MS_NAME}")
   }
   stage('Tagging Image for Dev')
   {
	sh'docker pull docker-registry.default.svc:5000/$APP_NAME-dev-apps/$MS_NAME'
	sh'docker tag  docker-registry.default.svc:5000/$APP_NAME-dev-apps/$MS_NAME:latest 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:dev-apps'
	sh'docker push 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:dev-apps'
	   
   }
   stage('Dev - Deploy Application')
   {
       sh 'oc apply -f Orchestration/deployment-dev.yaml -n=${APP_NAME}-dev-apps'
       sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-dev-apps'
       
   }
	
   stage('Tagging Image for Testing')
   { 
       sh'docker tag  13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:dev-apps 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:test-apps'
       sh'docker push 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:test-apps'
   }
   if(env.TESTING == 'True')
   {	
       stage('Test - Deploy Application')
       {
              sh 'oc apply -f Orchestration/deployment-test.yaml -n=${APP_NAME}-test-apps'
              sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-test-apps'
       }

       node('selenium')
       {

          stage('Integration Testing')
          {
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
               sh 'oc apply -f Orchestration/deployment-qa.yaml -n=${APP_NAME}-qa-apps'
               sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-qa-apps'
       }
       node('selenium')
       {
            stage('Integration Testing')
            {
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
        openshiftTag(namespace: '$APP_NAME-dev-apps', srcStream: '$MS_NAME', srcTag: 'latest', destStream: '$MS_NAME', destTag: 'pt-apps')
	  sh'docker tag  13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:test-apps 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:pt-apps'
       sh'docker push 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:pt-apps'  
   }
if(env.PT == 'True')
   {	

        stage('Test - Deploy Application')
         {
                sh 'oc apply -f Orchestration/deployment-pt.yaml -n=${APP_NAME}-pt-apps'
                sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-pt-apps'
         }

        stage('Performance Testing')
        {
                sh 'mvn verify'
        }
	     
    }

	stage('Tagging Image for UAT')
   	{
       		openshiftTag(namespace: '$APP_NAME-dev-apps', srcStream: '$MS_NAME', srcTag: 'latest', destStream: '$MS_NAME', destTag: 'uat-apps')
		sh'docker tag  13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:pt-apps 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:uat-apps'
       		sh'docker push 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:uat-apps'
   	}
	stage('Test - UAT Application')
	 {
		sh 'oc apply -f Orchestration/deployment-uat.yaml -n=${APP_NAME}-uat-apps'
       		sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-uat-apps'
	 }
	stage('Tagging Image for Pre-Prod')
   	{
       		openshiftTag(namespace: '$APP_NAME-dev-apps', srcStream: '$MS_NAME', srcTag: 'latest', destStream: '$MS_NAME', destTag: 'preprod-apps')
		sh'docker tag  13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:uat-apps 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:preprod-apps'
       		sh'docker push 13.89.217.239:5000/$APP_NAME-dev-apps/$MS_NAME:preprod-apps'
   	}
	stage('Test - Preprod Application')
	 {
		sh 'oc apply -f Orchestration/deployment-preprod.yaml -n=${APP_NAME}-preprod-apps'
       		sh 'oc apply -f Orchestration/service.yaml -n=${APP_NAME}-preprod-apps'
	 }
	}
	catch(e){
		echo "Pipeline has failed"
		emailext body: "${env.BUILD_URL} has result ${currentBuild.result}", subject: "Status of pipeline: ${currentBuild.fullDisplayName}", to: '${mailrecipent}'
	}
	
 
}
}	
