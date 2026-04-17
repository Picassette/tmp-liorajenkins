pipeline {
	// Agent selection
	agent any // No need for a particular agent
	
	// We delare the env here
	environment {
		// Note : DOCKER_USERNAME is also set as credential
		DOCKER_IMAGE_CAST = "examjenkins_cast"
		DOCKER_IMAGE_MOVIE = "examjenkins_movie"
		DOCKER_TAG = "v.${BUILD_NUMBER}.0"
        DOCKER_TEST_CONTAINER_MOVIE = ""
        DOCKER_TEST_CONTAINER_CAST = ""
	}
	
	// Declare stages
	stages {
		// Generate a random name used later in Docker Image testing
		stage('generate : random name for Docker Image testing') {
			steps {
				script {
					echo "Generating Docker Image Test name"

					def randomSuffix = UUID.randomUUID().toString().substring(0, 8)
					
                    env.DOCKER_TEST_CONTAINER_MOVIE = "movie_test_${randomSuffix}"
                    env.DOCKER_TEST_CONTAINER_CAST = "cast_test_${randomSuffix}"

					echo "Containers: ${env.DOCKER_TEST_CONTAINER_MOVIE}, ${env.DOCKER_TEST_CONTAINER_CAST}"
				}
			}
		}
		// Build our Docker images
		stage('run : Docker Build') {
			// We get our Docker Name (Scopped bc of good practice)
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					// Script run for Docker Image Build Movie
					
					sh '''
						echo "Building image : $DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG"
						docker build -t $DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
						echo "Image builded !"
					echo "Building image : $DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG"
						docker build -t $DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
						echo "Image builded !"
					'''
				}
			}
		}
		// Running our Docker images and check if it run correctly
		stage('run : run & test Docker Image') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					sh '''
						echo "Starting test container: $DOCKER_TEST_CONTAINER_MOVIE"
						
						docker run -d -p 8001:8001 --name $DOCKER_TEST_CONTAINER_MOVIE $DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
						
						echo "Waiting for service to be ready"
						
						# Retry 10 time max , with a 2s wait between each retry
						for i in {1..10}; do
							echo "Attempt $i/10"
							
							HTTP_CODE=$(curl -s -f -o /dev/null -w "%{http_code}" http://localhost:8001)
							
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
						docker stop $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true
						docker rm -f $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true
						exit 1
					'''
				}
			}
			steps {
				script {
					sh '''
						echo "Starting test container: $DOCKER_TEST_CONTAINER_CAST"
						
						docker run -d -p 8002:8002 --name $DOCKER_TEST_CONTAINER_CAST $DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG
						
						echo "Waiting for service to be ready"
						
						# Retry 10 time max , with a 2s wait between each retry
						for i in {1..10}; do
							echo "Attempt $i/10"
							
							HTTP_CODE=$(curl -s -f -o /dev/null -w "%{http_code}" http://localhost:8002)
							
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
						docker stop $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true
						docker rm -f $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true
						exit 1
					'''
				}
			}
		}
		// We remove the temporary build Docker Images
		stage('Cleanup') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
			}
			steps {
				script {
					sh '''
						echo "Cleaning up testing artifacts"
						
						# We stop the containers if they run
						docker stop $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true
						docker stop $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true

						# We remove the containers if they exist
						docker rm -f $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true
						docker rm -f $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true

						# We delete the test images
						docker rmi -f "$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG" || true
						docker rmi -f "$DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG" || true
						
						echo "Cleanup completed"
					'''
				}
			}
		}
		// We push the Docker Images
		stage('run : Pushing Docker Images') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				DOCKER_PASS = credentials("DOCKER_PASS")
			}
			steps {
				script {
					sh '''
						"$DOCKER_PASS" | docker login -u $DOCKER_USERNAME --password-stdin
						docker push $DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
						docker push $DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG
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
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE
						helm upgrade --install fastapi Chart \
                            --namespace dev \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST
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
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE
						helm upgrade --install fastapi Chart \
                            --namespace qa \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST
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
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE
						helm upgrade --install fastapi Chart \
                            --namespace staging \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST
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
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE
						helm upgrade --install fastapi Chart \
                            --namespace prod \
                            --set image.tag=$DOCKER_TAG \
                            --set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST
					'''
				}
			}
		}
	}
}