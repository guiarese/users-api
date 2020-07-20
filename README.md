# Projeto de Avaliação DevOps

## Integrantes
|NOME|RM|
| -------- | -------- | 
|GUILHERME ALMEIDA REZENDE|335477|
|JAQUELINE DE CARVALHO LAURENTI|335815|
|LUCAS GABRIEL DAMASCENA DE SOUZA|335698|
|SERGIO HENRIQUE PEDROSO OLIVEIRA|335231|
|WILLIAM DA SILVA ROCHA|335986|

## Modelo: GitHub Flow
A equipe é composta por cinco alunos com o objetivo de entregar um MVP (Minimum Viable Product) para certificação do MBA, então gostaríamos de organizar nosso projeto em workflow simples e que seja de fácil entendimento aos integrantes que não tenham a vivência dessa metodologia.

### Vantagens

* Amigável para CI/CD;
* Desenvolvimento baseado SEMPRE em relação a última aplicação enviada como release;
* Em relação ao GitFlow, possui menor complexidade e possibilita maior entendimento do histórico de projeto;
* O processo começa SEMPRE via pull request, possibilitando discussões e code review antes da alteração entrar na esteira;
* Praticidade de Implantação, Scott Chacon cita em artigo de Agosto, 2011: 
>"Se você implantar a cada poucas horas, é quase impossível introduzir um grande número de grandes bugs. Pequenos problemas podem ser introduzidos, mas podem ser corrigidos e reimplantados muito rapidamente. Normalmente, você teria que fazer um 'hotfix' ou algo fora do processo normal, mas isso é simplesmente parte do nosso processo normal - não há diferença no fluxo do GitHub entre um hotfix e um recurso muito pequeno."

### Desvantagens

* Em produção pode se tornar instável (Justamente pela regularidade de implantações citadas por Scott Chacon);
* Não existe convenção de nomes para branches. Então é uma questão que precisa ser elaborada e monitorada pela equipe para evitar desorganização.
* Por não existir **branch intermediária** o pull request é realizado diretamente na **branch master**;

### Cenários do Pipeline

O pipeline será capaz de cobrir as seguintes atividades:

* Pull Requests;
* Code Analysis;
* Testes;
* Merges;
* Deploy.
 
### Modelo de Implementação

Como a esteira tem início com pull request, significa que existe uma fase de validação/code review realizada pela equipe, o que permite a escolha somente do **Deployment Continuous**.

## Representação Gráfica

![Flowchart](../master/img/Graphic.jpeg)

- Usuário realiza pull request na branch master;
- Por meio de **Webhook** o Git aciona o pipeline do Jenkins;
- O **SonarQube** efetua a varredura de código verificando sua qualidade;
- O **Jenkins** efetua o **stop service** em produção; 
- Testes unitários são acionados e validados pelo framework NodeJS;
- Se aprovado, uma imagem é compilada e gravada no DockerHub | Se não aprovado, uma mensagem de erro é exibida e o pipeline é encerrado;
- Por fim, a imagem clonada pelo Docker da máquina e utiliza novamente o endereço **http://localhost:3000** para subir. 

**Em todas as etapas existe notificação ao Slack, por dois motivos: Registro de Log e Informação Atualizada a Todos**

### Script

* Seguimos a configuração em sala de aula com Docker Compose;
* Importante ressaltar que para o funcionamento do **DockerHub**, **Slack** e **Sonar**, se faz necessário configurações internas de credenciais no Jenkins;
* Estamos trabalhando com um projeto NodeJS;

```
pipeline {
    
    environment { 
        CI = "true"
        image = ''
        registryUrl = 'guiareze/users-api'
        registryCredentialsId = 'dockerhub_id';
    }
    
    agent any
    stages {

	stage("Initial configs") {
		steps {
			sh "docker ps -a -q | xargs -n 1 -P 8 -I {} docker stop {}"
			slackSend channel: '#users-api', 
                          message: 'Parando serviços ativos para a subida dos novos'
		}
	}
	    
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
			slackSend channel: '#users-api', 
                          message: 'Scanner do sonar já efetuado. Código sem problemas de duplicidade/vulnerabilidade/etc.'
    		}
	}
	    
        stage('Unit Tests') {
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
	    
        stage("DockerHub promotion") {
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
        
        stage('Delivery - run service in production') {
            steps {
                sh "docker run -d -p 3000:3000 $registryUrl:$BUILD_NUMBER"
		sh "docker ps"
		slackSend channel: '#users-api', 
                          message: 'Aplicação funcionando no caminho localhost:3000'
            }
        }        
    }
}
```
