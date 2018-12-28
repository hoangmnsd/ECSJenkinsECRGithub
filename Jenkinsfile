	import groovy.json.*
	import groovy.json.JsonOutput
	pipeline {
		agent any

	    environment {
	        VERSION = 'latest'
	        PROJECT = 'hello-world'
	        IMAGE = 'hello-world:latest'
	        ECRURL = 'https://856233045188.dkr.ecr.us-east-1.amazonaws.com/hello-world'
			REGION='us-east-1'
			REPOSITORY_NAME='hello-world'
			CLUSTER='getting-started'
			def FAMILY= sh (
				script: "sed -n \'s/.*\"family\": \"\\(.*\\)\",/\\1/p\' taskdef.json",
				returnStdout: true
				).trim()
			// echo "FAMILY: ${FAMILY}"
			def NAME= sh (
				script: "sed -n \'s/.*\"name\": \"\\(.*\\)\",/\\1/p\' taskdef.json",
				returnStdout: true
				).trim()
			// echo "NAME: ${NAME}"
			SERVICE_NAME="${NAME}-service"

	    }
		stages {
			stage('Build preparations') {
	            steps {
	                script {
	                    // calculate GIT lastest commit short-hash
	                    gitCommitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
	                    shortCommitHash = gitCommitHash.take(7)
	                    // calculate a sample version tag
	                    VERSION = shortCommitHash
	                    // set the build display name
	                    currentBuild.displayName = "#${BUILD_ID}-${VERSION}"
	                    IMAGE = "$PROJECT:$VERSION"
	                }
	            }
	        }
			stage('Docker build') {
	            steps {
 	                script {
 	                	//Get Login ECR
 	                	sh("eval \$(aws ecr get-login --no-include-email --region us-east-1 | sed 's|https://||')")
 	                    // Build the docker image using a Dockerfile
                    	docker.build("$IMAGE")
                	}
	            }
	        }
	        stage('Push image to ECR') {
	        	steps {
 	                script {
				        docker.withRegistry(ECRURL) {
                        	docker.image(IMAGE).push()
                    	}
        			}
    			}
    		}
    		stage('Deploy to ECS') {
	        	steps {
 	                script {
						//Store the repositoryUri as a variable
						def REPOSITORY_URI = sh (
							script: "aws ecr describe-repositories --repository-names ${REPOSITORY_NAME} --region ${REGION} | jq .repositories[].repositoryUri | tr -d \'\"\'",
							returnStdout: true
							).trim()
						echo "REPOSITORY_URI: ${REPOSITORY_URI}"
						//Replace the build number and respository URI placeholders with the constants above
						sh "sed -e \"s;%BUILD_NUMBER%;${VERSION};g\" -e \"s;%REPOSITORY_URI%;${REPOSITORY_URI};g\" taskdef.json > ${NAME}-v_${BUILD_NUMBER}.json"
						//Register the task definition in the repository
						sh "aws ecs register-task-definition --family ${FAMILY} --cli-input-json file://${WORKSPACE}/${NAME}-v_${BUILD_NUMBER}.json --region ${REGION}"
						def SERVICES= sh (
							script: "aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .failures[]",
							returnStdout: true
							).trim()
						echo "SERVICES: ${SERVICES}"
						//Get latest revision
						def REVISION= sh (
							script: "aws ecs describe-task-definition --task-definition ${NAME} --region ${REGION} | jq .taskDefinition.revision",
							returnStdout: true
							).trim()
						echo "REVISION: ${REVISION}"
						//Create or update service
							if ( "${SERVICES}" == "" ) {
							  echo "entered existing service"
							  def DESIRED_COUNT= sh (
							  	script: "aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .services[].desiredCount",
							  	returnStdout: true
							  	).trim()
							  if ( "${DESIRED_COUNT}" == "0" ) {
							    DESIRED_COUNT="1"
							  }
							  sh "aws ecs update-service --cluster ${CLUSTER} --region ${REGION} --service ${SERVICE_NAME} --task-definition ${FAMILY}:${REVISION} --desired-count ${DESIRED_COUNT}"
							} else {
							  echo "entered new service"
							  sh "aws ecs create-service --service-name ${SERVICE_NAME} --desired-count 1 --task-definition ${FAMILY} --cluster ${CLUSTER} --region ${REGION}"
							}
        			}
    			}
    		}
		}
	}
