import org.apache.commons.lang.StringUtils

def bldNamespace = "${ocpNamespaceBase}-bld"
def devNamespace = "${ocpNamespaceBase}-dev-01"
def tstNamespace = "${ocpNamespaceBase}-tst-01"

// Declare these here to explicitly show they have script scope
def artifact = "ips-microservice"
def devSecureroute = "${artifact}-dev.ocp-eu.testweb.bp.com"
def tstSecureroute = "${artifact}-tst.ocp-eu.testweb.bp.com"

node('centos-large') {

	stage ('Node App Build') {

    	git url: gitUrl, branch: gitBranch, credentialsId: gitCredentialsIdvsts
	
    stage ('Image Build') {

		withCredentials([usernamePassword(credentialsId: "${ocTokenCredentialsId}", passwordVariable: 'ocToken', usernameVariable: 'dummy')]) {
			sh "mkdir package-contents"
			sh "mv integration-test src Dockerfile package.json package-contents"
			sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} process -f ocp-build-template.yml -n ${bldNamespace} NAME=${artifact} IS_TAG=${artifact}.${BUILD_NUMBER} | oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} apply -n ${bldNamespace} -f -"
			sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} start-build ${artifact} --from-dir=package-contents/ --follow -n ${bldNamespace}"
		}
    }

    stage ('DEV Deploy') {

    	dir ('ocp-common') {
    		// Clone the common shared Ansible roles.
    		git url: gitCommonRolesUrl, branch: gitCommonRolesBranch, credentialsId: gitCredentialsId
    	}
		
		dir ('ansible-playbooks') {
			// Clone the app playbooks.
        	git url: gitConfigRepoUrl, branch: gitConfigRepoBranch, credentialsId: gitCredentialsIdvsts

			withCredentials([usernamePassword(credentialsId: "${vaultCredentialsId}", passwordVariable: 'vaultPw', usernameVariable: 'dummy')]) {
				sh "echo ${vaultPw} > vault.pw"
			}
			
        	withEnv(['ANSIBLE_ROLES_PATH=../ocp-common/roles']) {
       	    	withCredentials([usernamePassword(credentialsId: "${ocTokenCredentialsId}", passwordVariable: 'ocToken', usernameVariable: 'dummy')]) {
                    numReplicas = 1
					sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} delete configmap ${artifact}-config-1.0 -n ${devNamespace}  || true"
					sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} delete secret ${artifact}-secret-1.0 -n ${devNamespace} || true"
                    sh "ansible-playbook ocp-deploy-playbook.yml --extra-vars='{\"ocToken\":\"${ocToken}\", \"dcName\":\"${artifact}\", \"namespace\":\"${devNamespace}\", \"NamespaceBase\":\"${ocpNamespaceBase}\", \"bldNamespace\":\"${bldNamespace}\", \"numReplicas\":\"${numReplicas}\", \"isTag\":\"${artifact}.${BUILD_NUMBER}\", \"configVersion\":\"${configVersion}\", \"secureRoute\":\"${devSecureroute}\" }' --vault-password-file=vault.pw"
					openshiftVerifyDeployment apiURL: "${ocpUrl}", authToken: "${ocToken}", depCfg: "${artifact}", namespace: "${devNamespace}", replicaCount: "${numReplicas}", verifyReplicaCount: 'true', waitTime: '600', waitUnit: 'sec'
    			}
    		}
 		}
	}
	
   stage ('Running Unit Test') {
		withCredentials([usernamePassword(credentialsId: "${ocTokenCredentialsId}", passwordVariable: 'ocToken', usernameVariable: 'dummy')]) { 
			OC_EXEC="oc exec --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} -n ${devNamespace} "
			OC="oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} "

			sh """
			
			Pod=\$(${OC} get pods -n ${devNamespace} -o wide  | grep ${artifact} | grep Running |awk \'{print \$1}\')
			
			echo Running Unit test for Pod: \$Pod
			${OC_EXEC} \$Pod  -- bash -c \"npm run unit\" 

			"""
		//	junit 'report.xml'
		}
	} 
   

	stage ('Running Integration Test') { 
	//	build job: 'IPS Integration Test'

	}
   
	// Save the files that we've checked out so we can reuse them when we resume on a different node.
	stash "${ocpNamespaceBase}-pipeline"
}


stage ('TST Approval') {
	timeout(time:5, unit:'DAYS') {
   		input "Deploy to TST?"
	}
}

node('centos-medium') {

	stage ('TST Deploy') {
		// Restore the files that we saved away earlier.
		unstash "${ocpNamespaceBase}-pipeline"

		dir ('ansible-playbooks') {
        	withEnv(['ANSIBLE_ROLES_PATH=../ocp-common/roles']) {
       	    	withCredentials([usernamePassword(credentialsId: "${ocTokenCredentialsId}", passwordVariable: 'ocToken', usernameVariable: 'dummy')]) {
                    numReplicas = 1
					sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} delete configmap ${artifact}-config-1.0 -n ${tstNamespace} || true"
					sh "oc --server=${ocpUrl} --insecure-skip-tls-verify=true --token=${ocToken} delete secret ${artifact}-secret-1.0 -n ${tstNamespace} || true"
                    sh "ansible-playbook ocp-deploy-playbook.yml --extra-vars='{\"ocToken\":\"${ocToken}\", \"dcName\":\"${artifact}\", \"namespace\":\"${tstNamespace}\", \"NamespaceBase\":\"${ocpNamespaceBase}\", \"bldNamespace\":\"${bldNamespace}\", \"numReplicas\":\"${numReplicas}\", \"isTag\":\"${artifact}.${BUILD_NUMBER}\", \"configVersion\":\"${configVersion}\", \"secureRoute\":\"${tstSecureroute}\" }' --vault-password-file=vault.pw"
					openshiftVerifyDeployment apiURL: "${ocpUrl}", authToken: "${ocToken}", depCfg: "${artifact}", namespace: "${tstNamespace}", replicaCount: "${numReplicas}", verifyReplicaCount: 'true', waitTime: '600', waitUnit: 'sec'
    			}
    		}
 		}
	}
}
}
