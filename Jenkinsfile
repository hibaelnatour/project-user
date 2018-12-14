#!/usr/bin/env groovy

String gitEmail = "hibaelnatour@gmail.com"
String gitUser = "hibaelnatour"
String gitRepoUser_URL = "https://github.com/hibaelnatour/project-user.git"
String gitRepoAdmin_URL = "https://github.com/hibaelnatour/project-admin.git" 

String apiTransfoEndpoint = "http://api-transformation-myaccess-reg-jenk-poc.apps.na39.openshift.opentlc.com/api/v1/transfo"
String workspace
String userConfFilename
String finalConfFilename

def err = null
def response

currentBuild.result = "SUCCESS"

def toJson = {
	input ->
	groovy.json.JsonOutput.toJson(input)
}

try {

	node {

		workspace = pwd()

		//Checkout Git repo project-user 
		stage(name: 'Checkout'){
			sh 'git config --global user.email ${gitEmail}'
			sh 'git config --global user.name ${gitUser}'
			sh 'git config --global push.default simple'
			git url: gitRepoUser_URL, branch: 'master'
		}

		//Get userConfFilename in jenkins workspace after git checkout
		stage(name: 'Get file'){
			def files = findFiles(glob: 'project.json') 
			echo "${files[0].name}"    	
			userConfFilename = "${files[0].name}"
		}

		//Call API transformation for conf file gathering 
		stage(name: 'Transformation'){
			def data = readJSON file: userConfFilename
			
			echo toJson(data)
			
			response = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', httpMode: 'POST', requestBody: toJson(data), url: apiTransfoEndpoint, validResponseCodes: '200'
			
		}
		
		dir('ProjAdminCheckout') {
			//Checkout repo
			stage(name: "Chekout"){
				sh 'git config user.email ${gitEmail}'
				sh 'git config user.name ${gitUser}'
				git url: gitRepoAdmin_URL, branch: 'master'
			}
			
			//Get finalConfFilename in jenkins workspace after git checkout
			stage(name: "Get file"){
				def files = findFiles(glob: 'final.json') 
				echo "${files[0].name}"    	
				finalConfFilename = "${files[0].name}"
				writeFile file: finalConfFilename, text: response.content
			}
			
			stage(name: "push"){
				sh 'git add .'
				sh 'git commit -m "test commit jenkins"'
				sh 'git push --set-upstream https://hibaelnatour:Hemo2013@github.com/hibaelnatour/project-admin.git master'
				//sh 'git push https://hibaelnatour:Hemo2013@github.com/hibaelnatour/project-admin.git'
				sh 'git tag -a finalConf_v2 -m "Jenkins"'
				sh 'git push https://hibaelnatour:Hemo2013@github.com/hibaelnatour/project-admin.git --tags'
			}
		}
	}

} catch (caughtError) {

    err = caughtError
    currentBuild.result = "FAILURE"

}


