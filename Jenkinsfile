#!groovy

import groovy.json.JsonSlurperClassic


node {
    
 
    def SF_CONSUMER_KEY=env.SF_CONSUMER_KEY
    def SF_USERNAME=env.SF_USERNAME
    def SERVER_KEY_CREDENTALS_ID=env.SERVER_KEY_CREDENTALS_ID
    def TEST_LEVEL='RunLocalTests'
    def PACKAGE_NAME='0Ho2w000000PAxXCAW'
    def PACKAGE_VERSION


    def toolbelt = tool 'toolbelt'
    
    

    // -------------------------------------------------------------------------
    // Check out code from source control.
    // -------------------------------------------------------------------------

    stage('checkout source') {
       
        checkout scm 
       
    }


    // -------------------------------------------------------------------------
    // Run all the enclosed stages with access to the Salesforce
    // JWT key credentials.
    // -------------------------------------------------------------------------

    withCredentials([file(credentialsId: SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {

        // -------------------------------------------------------------------------
        // Authorize the Dev Hub org with JWT key and give it an alias.
        // -------------------------------------------------------------------------

        stage('Authorize DevHub') {
           rc = command "${toolbelt}\\sfdx force:auth:jwt:grant --clientid \"${SF_CONSUMER_KEY}\" --username \"${SF_USERNAME}\" --jwtkeyfile \"${server_key_file}\" --setdefaultdevhubusername --setalias DevHub"
            if (rc != 0) {
                error 'Salesforce dev hub org authorization failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Create new scratch org to test your code.
        // -------------------------------------------------------------------------

        stage('Create Test Scratch Org') {
            rc = command "${toolbelt}/sfdx force:org:create --targetdevhubusername DevHub --setdefaultusername --definitionfile config/project-scratch-def.json --setalias ciorg --wait 10 --durationdays 1"
            
            if (rc != 0) {
                error 'Salesforce test scratch org creation failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Display test scratch org info.
        // -------------------------------------------------------------------------

        stage('Display Test Scratch Org') {
            rc = command "${toolbelt}\\sfdx force:org:display --targetusername ciorg"
            if (rc != 0) {
                error 'Salesforce test scratch org display failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Push source to test scratch org.
        // -------------------------------------------------------------------------

        stage('Push To Test Scratch Org') {
            rc = command "${toolbelt}\\sfdx force:source:push --targetusername ciorg"
            if (rc != 0) {
                error 'Salesforce push to test scratch org failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Run unit tests in test scratch org.
        // -------------------------------------------------------------------------

        stage('Run Tests In Test Scratch Org') {
            rc = command "${toolbelt}\\sfdx force:apex:test:run --targetusername ciorg --wait 10 --resultformat tap --codecoverage --testlevel ${TEST_LEVEL}"
            if (rc != 0) {
                error 'Salesforce unit test run in test scratch org failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Delete test scratch org.
        // -------------------------------------------------------------------------

        stage('Delete Test Scratch Org') {
            rc = command "${toolbelt}\\sfdx force:org:delete --targetusername ciorg --noprompt"
            if (rc != 0) {
                error 'Salesforce test scratch org deletion failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Create package version.
        // -------------------------------------------------------------------------

        stage('Create Package Version') {
           
                    if (isUnix()) {
                        output = sh returnStdout: true, script: "${toolbelt}\\sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername DevHub"
                    } else {
                        output = bat(returnStdout: true, script: "${toolbelt}\\sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername DevHub").trim()
                        output = output.readLines().drop(1).join(" ")
            }


            // Wait 5 minutes for package replication.
            sleep 300
            
            try{
               // echo "Before json call"
             def jsonSlurper = new JsonSlurperClassic()
            def response = jsonSlurper.parseText(output)
//echo "Before Package creation"
            PACKAGE_VERSION = response.result.SubscriberPackageVersionId
            response = null

          //  echo ${PACKAGE_VERSION}
            }
            catch(err)
            {
                echo $err
                
            }
		    
          }
        // -------------------------------------------------------------------------
        // Authenticate Sandbox org to install package to.
        // -------------------------------------------------------------------------

        stage('Staging - Sandbox Org') {
          
            echo "Authenticate Sandbox Org to install package to"
            rc = command "${toolbelt}\\sfdx force:auth:sfdxurl:store -f package-sfdx-project.json -s -a myDevelopOrg"
            //rc = command "${toolbelt}\\sfdx force:org:create --targetdevhubusername DevHub --setdefaultusername --definitionfile config/project-scratch-def.json --setalias installorg --wait 10 --durationdays 1"
            if (rc != 0) {
                error 'Salesforce package install scratch org creation failed.'
            }
           
        }


       

        // -------------------------------------------------------------------------
        // Install package in Sandbox org.
        // -------------------------------------------------------------------------

        stage('Install Package In Sandbox Org') {
            
            rc = command "${toolbelt}\\sfdx force:package:install --targetusername myDevelopOrg --package ${PACKAGE_VERSION} --wait 10 --publishwait 10 --noprompt --json"
            // rc = command "${toolbelt}\\sfdx force:package:install --package ${PACKAGE_VERSION} --targetusername installorg --wait 10"
            if (rc != 0) {
                error 'Salesforce package install failed.'
            }
            }
        }    

    
    
}

def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}