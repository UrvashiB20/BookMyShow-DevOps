pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'urvashibhole/bookmyshow-app:1.0'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh '''
                        node --version
                        npm --version
                        npm ci --legacy-peer-deps
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./bookmyshow-app --disableYarnAudit --disableNodeAudit',
                    odcInstallation: 'DP-Check'
                )

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs --no-progress . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                        -t $DOCKER_IMAGE \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app
                '''
            }
        }

        stage('Docker Push') {
           steps {
                 withCredentials([
                 usernamePassword(
               	 credentialsId: 'docker',
                 usernameVariable: 'DOCKER_USERNAME',
                 passwordVariable: 'DOCKER_PASSWORD'
	              )
 	       ]) {
        	    sh '''
                	echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                	docker push "$DOCKER_IMAGE"
                	docker logout
            	'''
       			 }
	    }
	}

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "Current Kubernetes context:"
                    kubectl config current-context

                    echo "Applying Kubernetes deployment..."
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml

                    echo "Waiting for deployment..."
                    kubectl rollout status deployment/bms-app --timeout=180s

                    echo "Kubernetes resources:"
                    kubectl get pods
                    kubectl get svc
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'trivyfs.txt,**/dependency-check-report.xml',
                allowEmptyArchive: true
            )
        }

        success {
            echo 'BookMyShow CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'BookMyShow CI/CD pipeline failed. Check the stage logs.'
        }
    }
}
