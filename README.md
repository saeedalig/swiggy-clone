# Deploying  Swiggy clone app in Kubernetes Cluster using CICD Pipeline.

<div align="center">
  <img src="https://i.ytimg.com/vi/dMVrwaYojYs/hq720.jpg?sqp=-oaymwEhCK4FEIIDSFryq4qpAxMIARUAAAAAGAElAADIQj0AgKJD&rs=AOn4CLDFY-RWFRoPVJ8Cl1R4nXXC4ay1Ng" alt="Logo" width="90%" height="90%">
  <p align="center"</p>
</div>


### Pipeline Script

```groovy
def COLOR_MAP = [
    'FAILURE' : 'danger',
    'SUCCESS' : 'good'
]


pipeline {
    agent any

    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
	
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_USERNAME = "asa96"
        APP_NAME = "swiggy"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}" + "/" + "${APP_NAME}"
    }
				
    stages {
	
        stage('Cleanup Workspace'){
            steps {
                script {
                    cleanWs()
                }
            }
        }
		
        stage('Checkout SCM'){
            steps {
                git credentialsId: 'github-auth', 
                url: 'https://github.com/saeedalig/swiggy-clone.git',
                branch: 'main'
            }
        }

        stage('Code Analysis'){
            steps{
                withSonarQubeEnv('SonarCloud') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
					-Dsonar.organization=devopsas \
					-Dsonar.projectKey=swiggy \
					'''
                }
            }
        }

        stage("Quality Gate Status"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
		
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS Scan') {
            steps {
                sh "trivy fs . > trivy-fs.txt"
            }
        }

        stage("Docker Build & Tag"){
            steps{
                script{

                    sh "docker build -t ${APP_NAME} ."
                    sh "docker tag ${APP_NAME} ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker tag ${APP_NAME} ${IMAGE_NAME}:latest"
                    
                }
            }
        }

        stage("TRIVY Image Scan"){
            steps{
                sh "trivy image asa96/youtube:latest > trivy-image.txt" 
            }
        }

        stage('Docker Push') {

            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-auth', passwordVariable: 'PASSWD', usernameVariable: 'USER')]) {
                    sh "docker login -u ${env.USER} -p ${env.PASSWD}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${APP_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${APP_NAME}:latest"
                }
            }
        }

		
        stage('Delete Docker Images'){
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker rmi ${IMAGE_NAME}:latest"
            }
        }

        stage('Update k8s deployment file') {
			steps {
				script {
					// Navigate to manifest directory
					dir('kube-manifests') {
						// Display the content of the deployment.yml before modification
						sh "cat deployment.yml"

						// Update the image tag in deployment.yml
						sh "sed -i \"s|\\(image:.*${APP_NAME}\\).*|\\1:${BUILD_ID}|\" deployment.yml"

						// Display the content of the deployment.yml after modification
						sh "cat deployment.yml"
					}
				}
			}
		}
		
		stage('Push the changed deployment file to GitHub') {
			steps {
				script {
					// Configure Git user information
					sh """
						git config --global user.name "zeal"
						git config --global user.email "zeal@gmail.com"
					"""

					// Add the changed deployment file to the Git repository
					sh 'git add kube-manifests/deployment.yml'

					// Commit the changes
					sh 'git commit -m "Updated the deployment file"'

					// Push the changes to the GitHub repository
					withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
						sh 'git remote set-url origin https://$USER:$PASS@github.com/saeedalig/youtube-clone-app.git'
						sh 'git push origin main'
					}
				}
			}
		}
		post {
		always {
			emailext attachLog: true,
				subject: "'${currentBuild.result}'",
				body: "Project: ${env.JOB_NAME}<br/>" +
					"Build Number: ${env.BUILD_NUMBER}<br/>" +
					"URL: ${env.BUILD_URL}<br/>",
				to: 'zeal999@gmail.com',
				attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
			}
		}
	}
}

```

***You can take a look at the detailed "Server Setup for Pipeline" guide [here](https://github.com/saeedalig/Netflix-Clone-App).***






