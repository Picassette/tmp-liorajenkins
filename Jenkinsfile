pipeline {
	// Agent selection
	agent any // No need for a particular agent
	
	// We delare the env here
	environment {
		// Note : DOCKER_USERNAME is also set as credential
		DOCKER_IMAGE = "examjenkins"
		DOCKER_TAG = "v.${BUILD_NUMBER}.0"
		DOCKER_TEST_CONTAINER_NAME = ""
	}
	
	// Declare stages
	stages {
		// Generate a random name used later in Docker Image testing
		stage('generate : random name for Docker Image testing') {
			steps {
				script {
					echo "Generating Docker Image Test name"

					def randomSuffix = UUID.randomUUID().toString().substring(0, 8)
					
					env.DOCKER_TEST_CONTAINER_NAME = "jenkins_test_${randomSuffix}"

					echo "Generated Docker Image Test Container name : ${env.DOCKER_TEST_CONTAINER_NAME}"
				}
			}
		}
		// Build our Docker image
		stage('run : Docker Build') {
			// We get our Docker Name (Scopped bc of good practice)
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					// Script run for Docker Image Build
					
					sh '''
						echo "Building image : $DOCKER_USERNAME/$DOCKER_IMAGE:$DOCKER_TAG"
						docker build -t $DOCKER_USERNAME/$DOCKER_IMAGE:$DOCKER_TAG .
						echo "Image builded !"
					'''
				}
			}
		}
		// Running our Docker image and check if it run correctly
		stage('run : run & test Docker Image') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					sh '''
						echo "Starting test container: $DOCKER_TEST_CONTAINER_NAME"
						
						docker run -d -p 8080:80 --name $DOCKER_TEST_CONTAINER_NAME $DOCKER_NAME/$DOCKER_IMAGE:$DOCKER_TAG
						
						echo "Waiting for service to be ready"
						
						# Retry 10 time max , with a 2s wait between each retry
						for i in {1..10}; do
							echo "Attempt $i/10"
							
							HTTP_CODE=$(curl -s -f -o /dev/null -w "%{http_code}" http://localhost:8080)
							
							if [ "$HTTP_CODE" == "200" ]; then
								echo "Service responded correctly"
								exit 0
							else
								echo "Attempt $i failed with code: $HTTP_CODE"
								sleep 2
							fi
						done
						
						echo "Failure, service did not respond correctly after 10 attempts."

						# Cleanup
						docker stop $DOCKER_TEST_CONTAINER_NAME 2>/dev/null || true
						docker rm -f $DOCKER_TEST_CONTAINER_NAME 2>/dev/null || true
						exit 1
					'''
				}
			}
		}
		// We push the Docker Image
		stage('run : Pushing Docker Image') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				DOCKER_PASS = credentials("DOCKER_PASS")
			}
			steps {
				script {
					sh '''
						"$DOCKER_PASS" | docker login -u $DOCKER_USERNAME --password-stdin
						docker push $DOCKER_USERNAME/$DOCKER_IMAGE:$DOCKER_TAG
					'''
				}
			}
		}
		// We deploy on Dev env
		stage('deployment : Dev') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				KUBECONFIG = credentials("config")
			}
			steps {
				script {
					sh '''
						mkdir -p .kube

						cat $KUBECONFIG > .kube/config
						helm upgrade --install fastapi Chart \
                            --namespace dev \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_NAME/$DOCKER_IMAGE
					'''
				}
			}
		}
		// We deploy on QA env
		stage('deployment : QA') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				KUBECONFIG = credentials("config")
			}
			steps {
				script {
					sh '''
						mkdir -p .kube

						cat $KUBECONFIG > .kube/config
						helm upgrade --install fastapi Chart \
                            --namespace qa \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_NAME/$DOCKER_IMAGE
					'''
				}
			}
		}
		// We deploy on Staging env
		stage('deployment : Staging') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				KUBECONFIG = credentials("config")
			}
			steps {
				script {
					sh '''
						mkdir -p .kube

						cat $KUBECONFIG > .kube/config
						helm upgrade --install fastapi Chart \
                            --namespace staging \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_NAME/$DOCKER_IMAGE
					'''
				}
			}
		}
		// We deploy on Prod env
		stage('deployment : Prod') {

			// We only want to run this step if the current branch is Master
			when {
				branch 'master'
			}

			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				KUBECONFIG = credentials("config")
			}
			steps {
				script {
					sh '''
						mkdir -p .kube

						cat $KUBECONFIG > .kube/config
						helm upgrade --install fastapi Chart \
                            --namespace prod \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_NAME/$DOCKER_IMAGE
					'''
				}
			}
		}
		// We remove the temporary build Docker Image
		stage('Cleanup') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					sh '''
						echo "Cleaning up testing artifacts"
						
						# We stop the container if it run
						docker stop $DOCKER_TEST_CONTAINER_NAME 2>/dev/null || true

						# We remove the container if it exist
						docker rm -f $DOCKER_TEST_CONTAINER_NAME 2>/dev/null || true

						# We delete the test image
						docker rmi -f "$DOCKER_USERNAME/$DOCKER_IMAGE:$DOCKER_TAG" || true
						
						echo "Cleanup completed"
					'''
				}
			}
		}
	}
}