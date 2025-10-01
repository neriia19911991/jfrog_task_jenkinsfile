pipeline {
    agent any

    environment {
        GITHUB_CREDS = credentials('github-user')
        JFROG_TOKEN = credentials('JFrog-Token')
        JFROG_USER = credentials('JFrog-User')
        DOCKER_REPO = "jfhaneria"
        IMAGE_NAME = "jf"
        JF_URL = "https://trialgkeas1.jfrog.io/artifactory/api/docker"
        JFROG_CLI_LOG_LEVEL = "DEBUG"
        BUILD_NAME = "JFrog"
        BUILD_NUMBER = "${BUILD_NUMBER}"
        GITHUB_URL = "github.com/neriia19911991/jfrog_task.git"
        JFROG_BASE_URL = "trialgkeas1.jfrog.io"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Clone & Checkout') {
            steps {
                script {
                    currentBuild.displayName = "${IMAGE_NAME}-${params.version}-build-${env.BUILD_NUMBER}"
                }
                sh """
                git clone https://${GITHUB_CREDS}@${GITHUB_URL}
                cd jfrog_task
                git checkout ${params.version}
                """
            }
        }

        stage('Setup JFrog CLI') {
            steps {
                sh """
                curl -fL https://getcli.jfrog.io | sh
                chmod +x jfrog
                ./jfrog --version
                """
            }
        }

        stage('Configure JFrog Server') {
            steps {
                sh """
                jf c add jfrog-trial-server \
                  --url=https://${JFROG_BASE_URL} \
                  --user=${JFROG_USER} \
                  --access-token=${JFROG_TOKEN} \
                  --interactive=false
                jf c show jfrog-trial-server
                """
            }
        }



        stage('Install Dependencies & Tests') {
            when {
                // Expression to choose either to run the tests or not (requirement)
                expression { return params.RUN_TESTS == true }
            }
            steps {
                dir('jfrog_task') {
                    // Configure npm to use JFrog Artifactory
                    sh """
                        npm config set registry https://trialgkeas1.jfrog.io/artifactory/api/npm/npm-official/
                        npm config set //trialgkeas1.jfrog.io/artifactory/api/npm/npm-official/:_authToken=${JFROG_TOKEN}
                    """
                    // Install regular dependencies
                    sh "npm install"
                    
                    // Package and Publish the npm into Jfrog npm repo
                    sh "npm pack"
                    sh "jf npm-config --repo-resolve=npm --repo-deploy=npm-local"
                    sh "jf rt npm-publish ."

                    // Install jest-junit for reporting
                    sh "npm install --save-dev jest-junit"
        
                    // Ensure reports folder exists
                    sh "mkdir -p reports"
        
                    // Run lint and tests (optional based on your RUN_TESTS param)
                    sh """
                    if [ "${params.RUN_TESTS}" = "true" ]; then
                        npm run lint
                        NODE_ENV=test npx jest --detectOpenHandles --reporters=default --reporters=jest-junit --outputFile=reports/junit.xml
                    fi
                    """
                    // Publish JUnit report
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('Docker Login') {
            steps {
                // docker login to pull everything trough Jfrog platform
                sh """
                echo ${JFROG_TOKEN} | docker login ${JF_URL} -u ${JFROG_USER} --password-stdin
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('jfrog_task') {
                    // Docker build with option to run test in the docker or not (more info in the HA pdf)
                    sh """
                    docker build --build-arg RUN_TESTS=${params.RUN_TESTS} -t ${IMAGE_NAME}:${params.version} .
                    """
                }
            }
        }

        stage('Scan Docker Image & Generate SBOM and metadata') {
            steps {
                sh """
                jf --version
                docker save --output jfForScan.tar ${IMAGE_NAME}:${params.version}
                echo saved image into tar for scanning , start scanning:
                jf scan jfForScan.tar --format=simple-json > sbom.json
                echo "${JF_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${params.version}" > metadata.json
                """
                // Publish the SBOM to be shown with the current Jenkins build
                archiveArtifacts artifacts: 'sbom.json', fingerprint: true
            }
        }

        stage('Push Docker Image & metadata to Artifactory') {
            steps {
                // Tag and push for both the current version , and latest
                sh """
                echo check existing docker images
                docker images | grep ${IMAGE_NAME}

                docker tag ${IMAGE_NAME}:${params.version} ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${params.version}
                docker tag ${IMAGE_NAME}:${params.version} ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:latest
                docker push ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${params.version}
                docker push ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:latest

                # Save digest with the tag included
                docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE_NAME}:${params.version} | sed "s/@/:${params.version}@/" > metadata.json

                jf rt build-docker-create ${DOCKER_REPO} --image-file metadata.json --build-name=${BUILD_NAME} --build-number=${BUILD_NUMBER}
                """
            }
        }

        stage('Publish Build Info') {
            steps {
                dir('jfrog_task') {
                    // add build info (required)
                    sh """
                    jf rt bce ${BUILD_NAME} ${BUILD_NUMBER}
                    jf rt build-add-git ${BUILD_NAME} ${BUILD_NUMBER}
                    jf rt build-publish ${BUILD_NAME} ${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    post {
        always {
            // remove jfrog connection & clean images
            sh "jf c rm jfrog-trial-server || true"
            sh """
            docker rmi ${IMAGE_NAME}:${params.version} || true
            docker rmi ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${params.version} || true
            docker rmi ${JFROG_BASE_URL}/${DOCKER_REPO}/${IMAGE_NAME}:latest || true
            """
        }
    }
}
