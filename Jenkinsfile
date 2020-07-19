pipeline {
    
    environment { 
        CI = "true"
        image = ''
        registryUrl = 'guiareze/users-api'
        registryCredentialsId = 'dockerhub_id';
    }
    
    agent any
    stages {
	    
	stage('Sonarqube') {
    environment {
        scannerHome = tool 'sonarqubescanner'
    }
    steps {
        withSonarQubeEnv('sonarqube') {
            sh "${scannerHome}/bin/sonar-scanner"
        }
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}

	stage("Initial configs") {
		steps {
			sh "docker ps -a -q | xargs -n 1 -P 8 -I {} docker stop {}"
			slackSend channel: '#users-api', 
                          message: 'Parando serviços ativos para a subida dos novos'
		}
	}
	    
        stage('Test') {
            agent{
                docker {
                    image 'node:12-alpine'
                    args '-p 3000:3000 -p 5000:5000'
                }
            }
            
            steps {
                sh "npm install"
                sh "npm test"   
		slackSend channel: '#users-api', 
                          message: 'Testes unitários efetuados com sucesso'
            }
        }
	    
        stage("DockerHub Connection") {
            steps {
                echo "Init Clone Process"
                git  "https://github.com/guiarese/users-api.git"

            script {
                echo "Build Image"
                image = docker.build registryUrl + ":$BUILD_NUMBER"
                
                echo "Deploy on DockerHub"
                docker.withRegistry('', registryCredentialsId) {
					image.push()
				}
            	}    

                echo "Cleaning Up"
                sh "docker rmi $registryUrl:$BUILD_NUMBER"
		    
		slackSend channel: '#users-api', 
                          message: 'Deploy do container efetuado no docker hub'
		    
            }
        }
        
        stage('Deliver') {
            steps {
                sh "docker run -d -p 3000:3000 $registryUrl:$BUILD_NUMBER"
		sh "docker ps"
		slackSend channel: '#users-api', 
                          message: 'Aplicação funcionando no caminho localhost:3000'
            }
        }        
    }
}
