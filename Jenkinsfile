#!/usr/bin/env groovy

import java.util.Date


String gitEmail = "hibaelnatour@gmail.com"
String gitUser = "hibaelnatour"
String gitRepoUser_URL = "https://github.com/hibaelnatour/project-user.git"
String gitRepoAdmin_URL = "https://github.com/hibaelnatour/project-admin.git" 
String apiTransfoEndpoint = "http://api-transformation-myaccess-reg-jenk-poc.apps.na39.openshift.opentlc.com/api/v1/transfo"
String workspace
String userConfFilename
String finalConfFilename

def start = new Date()
def err = null
def response

currentBuild.result = "SUCCESS"

try {

	node {

		workspace = pwd()

		// clean up previous build
		stage (name : 'Cleanup') {
			//sh "rm -rf ${workspace}/env" à VOIR
		}

		//Checkout Git repo project-user 
		stage(name: 'Checkout'){
			sh 'git config user.email ${gitEmail}'
			sh 'git config user.name ${gitUser}'
			git url: ${gitRepoUser_URL}, branch: 'master'
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
			
			response = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', httpMode: 'POST', requestBody: toJson(data), url: ${apiTransfoEndpoint}, validResponseCodes: '200'
			

			//echo "..................."
			//sh "value=`cat test-conf.json`"
		}
		
		dir('ProjAdminCheckout') {
			//Checkout repo
			steps{
				sh 'git config user.email ${gitEmail}'
				sh 'git config user.name ${gitUser}'
				git url: ${gitRepoAdmin_URL}, branch: 'master'
			}
			
			//Get finalConfFilename in jenkins workspace after git checkout
			steps{
				def files = findFiles(glob: 'final.json') 
				echo "${files[0].name}"    	
				finalConfFilename = "${files[0].name}"
				//if (response.status == 200) {
					writeFile file: finalConfFilename, text: response.content
				//} else {
					//currentBuild.result = "FAILURE"
				//}
			}
			
			steps{
				sh 'git add .'
				sh 'git commit -m "test commit jenkins"'
				sh 'git push https://hibaelnatour:Hemo2013@github.com/hibaelnatour/project-admin.git'
				sh 'git tag -a finalConf_v1 -m "Jenkins tag"'
				sh 'git push https://hibaelnatour:Hemo2013@github.com/hibaelnatour/project-admin.git --tags'
			}
		}
	}

} catch (caughtError) {

    err = caughtError
    currentBuild.result = "FAILURE"

} finally {

    timeSpent = "\nTime spent: ${timeDiff(start)}"

    if (err) {
        throw err
    } else {
		currentBuild.result = "SUCCESS"
    }
}

def timeDiff(st) {
    def delta = (new Date()).getTime() - st.getTime()
    def seconds = delta.intdiv(1000) % 60
    def minutes = delta.intdiv(60 * 1000) % 60

    return "${minutes} min ${seconds} sec"
}

@NonCPS
def toJson = {
	input ->
	groovy.json.JsonOutput.toJson(input)
}
