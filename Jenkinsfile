pipeline {
	// Agent selection
	agent any // No need for a particular agent
	
	// We delare the env here
	environment {
		// Note : DOCKER_USERNAME & DOCKER_PASSWORD are set as credentials
		DOCKER_IMAGE_CAST = "examjenkins_cast"
		DOCKER_IMAGE_MOVIE = "examjenkins_movie"
		DOCKER_TAG = "v.${BUILD_NUMBER}.0"
		DOCKER_TEST_CONTAINER_MOVIE = "movie-test-${BUILD_NUMBER}"
		DOCKER_TEST_CONTAINER_CAST = "cast-test-${BUILD_NUMBER}"
		DOCKER_TEST_DB_MOVIE = "movie-db-test-${BUILD_NUMBER}"
		DOCKER_TEST_DB_CAST = "cast-db-test-${BUILD_NUMBER}"
		DOCKER_TEST_NETWORK = "test-net-${BUILD_NUMBER}"
	}
	
	// Declare stages
	stages {
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
						set -e

						# Creating a test network for our containers
						echo "Creating test network: $DOCKER_TEST_NETWORK"
						docker network create $DOCKER_TEST_NETWORK

						echo "Starting test db container: $DOCKER_TEST_DB_MOVIE"
						docker run -d --name $DOCKER_TEST_DB_MOVIE --network $DOCKER_TEST_NETWORK \
							-e POSTGRES_USER=movie_db_username \
							-e POSTGRES_PASSWORD=movie_db_password \
							-e POSTGRES_DB=movie_db_dev \
							postgres:12.1-alpine

						echo "Waiting for movie test database to be ready"
						for i in $(seq 1 15); do
							if docker exec $DOCKER_TEST_DB_MOVIE pg_isready -U movie_db_username -d movie_db_dev >/dev/null 2>&1; then
								echo "Movie test database is ready"
								break
						fi
							echo "Movie test database not ready yet ($i/15)"
							sleep 2
							if [ "$i" = "15" ]; then
								docker logs $DOCKER_TEST_DB_MOVIE || true
								exit 1
							fi
						done

						echo "Starting test container: $DOCKER_TEST_CONTAINER_MOVIE"
						
						docker run -d -p 8001:8000 --name $DOCKER_TEST_CONTAINER_MOVIE --network $DOCKER_TEST_NETWORK \
							-e DATABASE_URI=postgresql://movie_db_username:movie_db_password@$DOCKER_TEST_DB_MOVIE/movie_db_dev \
							-e CAST_SERVICE_HOST_URL=http://$DOCKER_TEST_CONTAINER_CAST:8000/api/v1/casts/ \
							$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG \
							uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio
						
						echo "Waiting for service to be ready"
						
						# Retry 10 time max , with a 2s wait between each retry
						success=0
						for i in $(seq 1 10); do
							echo "Attempt $i/10"
							
							# Check the OpenAPI endpoint to confirm the service is up.
							HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/api/v1/movies/openapi.json || true)
							if [ -z "$HTTP_CODE" ]; then HTTP_CODE="000"; fi
							
							if [ "$HTTP_CODE" = "200" ]; then
								echo "Service responded correctly"
								success=1
								break
							else
								if ! docker inspect $DOCKER_TEST_CONTAINER_MOVIE >/dev/null 2>&1; then
									echo "Movie test container no longer exists"
									exit 1
								fi
								if [ "$(docker inspect -f '{{.State.Running}}' $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true)" != "true" ]; then
									echo "Movie test container stopped unexpectedly"
									docker logs $DOCKER_TEST_CONTAINER_MOVIE || true
									exit 1
								fi
								echo "Attempt $i failed with code: $HTTP_CODE"
								sleep 2
							fi
						done

						if [ "$success" -ne 1 ]; then
							echo "Failure, movie service did not respond correctly after 10 attempts."
							docker logs $DOCKER_TEST_CONTAINER_MOVIE || true
							exit 1
						fi
					'''
				}
				script {
					sh '''
						set -e

						echo "Starting test db container: $DOCKER_TEST_DB_CAST"
						docker run -d --name $DOCKER_TEST_DB_CAST --network $DOCKER_TEST_NETWORK \
							-e POSTGRES_USER=cast_db_username \
							-e POSTGRES_PASSWORD=cast_db_password \
							-e POSTGRES_DB=cast_db_dev \
							postgres:12.1-alpine

						echo "Waiting for cast test database to be ready"
						for i in $(seq 1 15); do
							if docker exec $DOCKER_TEST_DB_CAST pg_isready -U cast_db_username -d cast_db_dev >/dev/null 2>&1; then
								echo "Cast test database is ready"
								break
						fi
							echo "Cast test database not ready yet ($i/15)"
							sleep 2
							if [ "$i" = "15" ]; then
								docker logs $DOCKER_TEST_DB_CAST || true
								exit 1
							fi
						done
						
						echo "Starting test container: $DOCKER_TEST_CONTAINER_CAST"
						
						docker run -d -p 8002:8000 --name $DOCKER_TEST_CONTAINER_CAST --network $DOCKER_TEST_NETWORK \
							-e DATABASE_URI=postgresql://cast_db_username:cast_db_password@$DOCKER_TEST_DB_CAST/cast_db_dev \
							$DOCKER_USERNAME/$DOCKER_IMAGE_CAST:$DOCKER_TAG \
							uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio
						
						echo "Waiting for service to be ready"
						
						# Retry 10 time max , with a 2s wait between each retry
						success=0
						for i in $(seq 1 10); do
							echo "Attempt $i/10"
							
							# Check the OpenAPI endpoint to confirm the service is up.
							HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8002/api/v1/casts/openapi.json || true)
							if [ -z "$HTTP_CODE" ]; then HTTP_CODE="000"; fi
							
							if [ "$HTTP_CODE" = "200" ]; then
								echo "Service responded correctly"
								success=1
								break
							else
								if ! docker inspect $DOCKER_TEST_CONTAINER_CAST >/dev/null 2>&1; then
									echo "Cast test container no longer exists"
									exit 1
								fi
								if [ "$(docker inspect -f '{{.State.Running}}' $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true)" != "true" ]; then
									echo "Cast test container stopped unexpectedly"
									docker logs $DOCKER_TEST_CONTAINER_CAST || true
									exit 1
								fi
								echo "Attempt $i failed with code: $HTTP_CODE"
								sleep 2
							fi
						done

						if [ "$success" -ne 1 ]; then
							echo "Failure, cast service did not respond correctly after 10 attempts."
							docker logs $DOCKER_TEST_CONTAINER_CAST || true
							exit 1
						fi
					'''
				}
			}
		}
		// We clean temporary test resources (Containers, network but not the images we will push on the registry)
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
						docker stop $DOCKER_TEST_DB_MOVIE 2>/dev/null || true
						docker stop $DOCKER_TEST_DB_CAST 2>/dev/null || true

						# We remove the containers if they exist
						docker rm -f $DOCKER_TEST_CONTAINER_MOVIE 2>/dev/null || true
						docker rm -f $DOCKER_TEST_CONTAINER_CAST 2>/dev/null || true
						docker rm -f $DOCKER_TEST_DB_MOVIE 2>/dev/null || true
						docker rm -f $DOCKER_TEST_DB_CAST 2>/dev/null || true

						# We remove the network if it exists
						docker network rm $DOCKER_TEST_NETWORK 2>/dev/null || true

						echo "Cleanup completed"
					'''
				}
			}
		}
		// We push the Docker Images
		stage('run : Pushing Docker Images') {
			environment {
				DOCKER_USERNAME = credentials("DOCKER_USERNAME")
				DOCKER_PASSWORD = credentials("DOCKER_PASSWORD")
			}
			steps {
				script {
					sh '''
						echo "$DOCKER_PASSWORD" | docker login -u $DOCKER_USERNAME --password-stdin
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
							helm upgrade --install fastapi-movie ./charts \
                            --namespace dev \
                            --create-namespace \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE \
							--set service.nodePort=30011 # We set different nodePort for each env to avoid conflicts
							helm upgrade --install fastapi-cast ./charts \
                            --namespace dev \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST \
							--set service.nodePort=30012
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
							helm upgrade --install fastapi-movie ./charts \
                            --namespace qa \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE \
							--set service.nodePort=30013
							helm upgrade --install fastapi-cast ./charts \
                            --namespace qa \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST \
							--set service.nodePort=30014
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
							helm upgrade --install fastapi-movie ./charts \
                            --namespace staging \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE \
							--set service.nodePort=30015
							helm upgrade --install fastapi-cast ./charts \
                            --namespace staging \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST \
							--set service.nodePort=30016
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

			input {
				message 'Deploy to production?'
				ok 'Yes'
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
							helm upgrade --install fastapi-movie ./charts \
                            --namespace prod \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_MOVIE \
							--set service.nodePort=30017
							helm upgrade --install fastapi-cast ./charts \
                            --namespace prod \
                            --set image.tag=$DOCKER_TAG \
							--set image.repository=$DOCKER_USERNAME/$DOCKER_IMAGE_CAST \
							--set service.nodePort=30018
					'''
				}
			}
		}
	}
}