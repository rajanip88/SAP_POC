pipeline {
  agent any

  //Configure the following environment variables before executing the Jenkins Job
  environment {
    IntegrationFlowID = "IntegrationFlow1"
    GetEndpoint = true //If you don't need the endpoint or the artefact does not provide an endpoint, set the value to false
    DeploymentCheckRetryCounter = 20 //multiply by 3 to get the maximum deployment time
	  CPIHost = "4858fb22trial.it-cpitrial04.cfapps.eu10-002.hana.ondemand.com"
	  
	  CPIOAuthHost = "4858fb22trial.authentication.eu10.hana.ondemand.com"
	  CPIOAuthCredentials = "f06c0988-4679-4ad3-b938-720cf3e30ce3$9xmIdLa0MOgVqZMWVdnUAnNSPymwjRI1-efnkeH9gvY="
  }

  stages {
    stage('Generate oauth bearer token') {
      steps {
        script {
          //get oauth token for Cloud Integration
          println("requesting oauth token");
			println("I entered my test value");
		println("env.CPIOAuthHost" + CPIHost);
			println("CPIOAuthHost"+CPIOAuthHost);
		
			println("env.CPIOAuthCredentials"+CPIOAuthCredentials);
          def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON',
            authentication: "${env.CPIOAuthCredentials}",
            contentType: 'APPLICATION_JSON',
            httpMode: 'POST',
            responseHandle: 'LEAVE_OPEN',
            timeout: 30,
            url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
          def jsonObjToken = readJSON text: getTokenResp.content
          def token = "Bearer " + jsonObjToken.access_token
          env.token = token
          getTokenResp.close();
        }
      }
    }

    stage('Deploy integration flow and check for deployment success') {
      steps {
        script {
          //deploy integration flow as specified in the configuration
          println("Deploying integration flow.");
          def deployResp = httpRequest httpMode: 'POST',
            customHeaders: [
              [maskValue: false, name: 'Authorization', value: env.token]
            ],
            ignoreSslErrors: true,
            timeout: 30,
            url: 'https://' + "${env.CPIHost}" + '/api/v1/DeployIntegrationDesigntimeArtifact?Id=\'' + "${env.IntegrationFlowID}" + '\'&Version=\'active\'';

          //check deployment status
          println("Start checking integration flow deployment status.");
          Integer counter = 0;
          def deploymentStatus;
          def continueLoop = true;
		  
		      //check until max check counter reached or we have a final state
          while (counter < env.DeploymentCheckRetryCounter.toInteger() & continueLoop == true) {
            Thread.sleep(3000);
            counter = counter + 1;
            def statusResp = httpRequest acceptType: 'APPLICATION_JSON',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: env.token]
              ],
              httpMode: 'GET',
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + "${env.CPIHost}" + '/api/v1/IntegrationRuntimeArtifacts(\'' + "${env.IntegrationFlowID}" + '\')';
            def jsonObj = readJSON text: statusResp.content;
            deploymentStatus = jsonObj.d.Status;

            println("Deployment status: " + deploymentStatus);
			
            if (deploymentStatus.equalsIgnoreCase("Error")) {
              //in case of error, get the error details
              def deploymentErrorResp = httpRequest acceptType: 'APPLICATION_JSON',
                customHeaders: [
                  [maskValue: false, name: 'Authorization', value: env.token]
                ],
                httpMode: 'GET',
                responseHandle: 'LEAVE_OPEN',
                timeout: 30,
                url: 'https://' + "${env.CPIHost}" + '/api/v1/IntegrationRuntimeArtifacts(\'' + "${env.IntegrationFlowID}" + '\')' + '/ErrorInformation/$value';
              def jsonErrObj = readJSON text: deploymentErrorResp.content
              def deployErrorInfo = jsonErrObj.parameter;
              println("Error Details: " + deployErrorInfo);
              statusResp.close();
              deploymentErrorResp.close();
              //End the whole job
              sh 'exit 1';
            } else if (deploymentStatus.equalsIgnoreCase("Started")) {
			        //Final status reached 
              println("Integration flow deployment successful")
              statusResp.close();
              continueLoop = false
            } else {
			        //Continue checking 
              println("The integration flow is not yet started. Will wait 3s and then check again.")
            }
          }
		      //After exiting the loop, react to the deployment state
          if (!deploymentStatus.equalsIgnoreCase("Started")) {
		        //If status not is Started, end the pipeline.
            println("No final deployment status reached. Current status: \'" + deploymentStatus);
            sh 'exit 1';
          } else {
            if (env.GetEndpoint.equalsIgnoreCase("true")) {
              //Get endpoint as configured above
              def endpointResp = httpRequest acceptType: 'APPLICATION_JSON',
                customHeaders: [
                  [maskValue: false, name: 'Authorization', value: env.token]
                ],
                httpMode: 'GET',
                responseHandle: 'LEAVE_OPEN',
                timeout: 30,
                url: 'https://' + "${env.CPIHost}" + '/api/v1/ServiceEndpoints?$filter=Name%20eq%20\'' + "${env.IntegrationFlowID}" + '\'&$select=EntryPoints&$format=json&$expand=EntryPoints'
              def jsonEndpointObj = readJSON text: endpointResp.content;
              def endpoint = jsonEndpointObj.d.results.EntryPoints.results.Url;
              def size = (jsonEndpointObj.d.results.EntryPoints.results).size();
              endpointResp.close();
			        //check if the flow has an endpoint
              if (size != 0) {
                println("Endpoint: " + endpoint);
              } else {
                unstable("The specified integration flow does not have an endpoint. Please check the flow or tenant")
              }
            }
          }
        }
      }
    }
  }
}
