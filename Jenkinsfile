
def FDC_RESULT_1
def FDC_RESULT_2
def CHANGED_PKGS
def RELEASED
def NOT_RELEASED

pipeline {
    agent any
    stages {
        
        stage('Rebuild and Check Cohort'){
            steps{
                 build job: 'Rebuild_and_Check', parameters: [string(name: 'PACKAGES_FILE', value: "$PACKAGES_FILE"), text(name: 'PACKAGES', value: ''), string(name: 'RELEASE', value: 'uat')]
                }
        }
     
        stage('Find Deployable Cohort') { 
            steps{
                build job: 'Find_Deployable_Cohort', parameters: [string(name: 'FROM_MODULE', value: 'R/uat'), string(name: 'TO_MODULE', value: 'R/prd'), string(name: 'PACKAGES_FILE', value: "$PACKAGES_FILE")]
                load "/jenkins/jenkins-home/workspace/Find_Deployable_Cohort/injected_variables.properties"
                script {
                    FDC_RESULT_1 = "${FDC_OUTPUT}"
                    if ("${FDC_CHANGED}" != "") {CHANGED_PKGS = "${FDC_CHANGED}"} else {CHANGED_PKGS = "No changed packages to release"}
                    if ("${FDC_OUTPUT}" != null){
                        if ("${FDC_OUTPUT}" != 'All packages passed'){
                            slackSend(
                                color: 'warning', \
                                message: "\n${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)\n Find Deployable Cohort completed with failed packages. \n Rebuilding and retrying the following failed packages: ${FDC_RESULT_1}" )
                        }
                    }
                }
            }
        }
     
        stage('Rebuild failed packages and Retry FDC'){
            when {expression {FDC_RESULT_1 != "All packages passed"}}
            steps{
                echo "Failed packages on Find Deployable Cohort - Rebuilding and Retrying the following packages: $FDC_RESULT_1"
                build job: 'Rebuild_GRAN_Packages', parameters: [string(name: 'PACKAGES_FILE', value: '/jenkins/jenkins-home/workspace/Find_Deployable_Cohort/fdc_failed_pkgs.txt'), text(name: 'PACKAGES', value: ''), string(name: 'RELEASE', value: 'uat')]
                build job: 'Find_Deployable_Cohort', parameters: [string(name: 'FROM_MODULE', value: 'R/uat'), string(name: 'TO_MODULE', value: 'R/prd'), string(name: 'PACKAGES_FILE', value: "$PACKAGES_FILE")]
                load "/jenkins/jenkins-home/workspace/Find_Deployable_Cohort/injected_variables.properties"
                script {
                    FDC_RESULT_2 = "${FDC_OUTPUT}"
                    if ("${FDC_CHANGED}" != ""){
                        CHANGED_PKGS = "${FDC_CHANGED}"
                        } else {
                            CHANGED_PKGS = "No changed packages to release"
                            }
                    echo "Changed packages: $CHANGED_PKGS"
                    if ("${FDC_OUTPUT}" != null){
                        if ("${FDC_OUTPUT}" != 'All packages passed'){
                            slackSend(
                                color: 'danger', \
                                message: "${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)\nSecond run of FDC completed with package failures. No packages will be released to R/prd. \n \n*LIST OF FAILED PACKAGES: ${FDC_OUTPUT}*\n \n*CHANGED PACKAGES NOT RELEASED TO PRD: ${CHANGED_PKGS}*" )
                        }
                    }
                }
            }
        }
        
     
        stage('No packages to release'){
            when{
                expression {CHANGED_PKGS == "No changed packages to release"}
                anyOf {expression { FDC_RESULT_2 == 'All packages passed'}; expression { FDC_RESULT_1 == 'All packages passed'}}
            }
            steps{
                echo "FDC completed with success, no changed packages to push"
                slackSend(
                    color: 'good', \
                    message: "\n${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)\n Completed with Success \n All packages passed FDC :smile: \n\nNo changed packages to release to R/prd" )
            }
        }
     
        stage('Release Packages'){
            when { 
                expression {CHANGED_PKGS != "No changed packages to release"}
                anyOf {expression { FDC_RESULT_2 == 'All packages passed'}; expression { FDC_RESULT_1 == 'All packages passed'}}
            }
            steps{
                echo "Changed Packages to be released to R/prd: $CHANGED_PKGS"
                build job: 'Release_Packages', parameters: [string(name: 'FROM_MODULE', value: 'R/uat'), string(name: 'TO_MODULE', value: 'R/prd'), text(name: 'TO_INSTALL', value: ''), string(name: 'PACKAGES_FILE', value: '/jenkins/jenkins-home/workspace/Find_Deployable_Cohort/changed_pkgs.txt'), string(name: 'EXCLUSION_PATH', value: '/foo/exclude_from_prd')]
                
                // The following lines are to account for any packages excluded from the release to production
                load "/jenkins/jenkins-home/workspace/Release_Packages/released_packages.txt"
                script{
                    if ("${RELEASED_PKGS}" != "") {RELEASED = "${RELEASED_PKGS}"} else {RELEASED = "No changed packages to release"}
                    NOT_RELEASED = "${FAILED_PKGS}"
                }
            }
            post{
                failure{
                    slackSend(
                        color: 'danger', \
                        message: "*WARNING: Release Packages Failed* \n${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)\n \n*CHANGED PACKAGES NOT RELEASED TO PRD: ${NOT_RELEASED}*" )
                }
                success{
                    slackSend(
                        color: 'good', \
                        message: "\n${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)\n Completed with Success \n All packages passed FDC :smile: \n\nChanged Packages released to R/prd: ${RELEASED}" ) 
                    build job: 'Sync_Native_and_Container_Environments', parameters: [string(name: 'ORIGIN_ENV', value: ''), string(name: 'ROSASLIND_DESTINATION_ENV', value: ''), booleanParam(name: 'CONTAINER', value: true)]
                }
            }
        }
    }    

    post {
         failure{
             slackSend(
                 color: 'danger', \
                 message: "*ERROR: Pipeline Failed* \n${env.JOB_NAME} - #${env.BUILD_NUMBER} (<${env.BUILD_URL}/console|Console Output>)" )
        }
    }
}
