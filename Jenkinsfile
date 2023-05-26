appName = "springbootsra"

pipeline {
	agent {
		label 'maven'
	}

	stages {
		stage('Build App') {
			steps {
				git branch: 'main', url: 'https://github.com/footfix73/springbootsra.git'
				
				script {
					def pom = readMavenPom file: 'pom.xml'
					version = pom.version
				}
				sh "mvn clean install -DskipTests=true"
            }
		}
		
		stage('Test'){
			steps{
				echo "Test Stage"
				//sh "${mvnCmd} test -Dspring.profiles.active=test"
				//step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
		}
		
		stage('Code Analysis'){
			steps{
				script{
					echo "Code Analysis"
					//sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000  -DskipTests=true"
				}
            }
        }
		
		stage('Create Image Builder'){
			when {
				expression {
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							return !openshift.selector("bc", "springbootsra").exists();
						}
					}
				}
            }
            steps {
				script {
					echo "Create Image Builder"
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.newBuild("--name=springbootsra", "--docker-image=registry.redhat.io/jboss-eap-7/eap74-openjdk11-openshift-rhel8", "--binary") 
						}
					}
				}
            }
		}
		
		stage('Build Image') {
			steps {
				sh "rm -rf ocp && mkdir -p ocp/deployments"
				sh "pwd && ls -la target "
				sh "cp target/spring-boot-*.jar ocp/deployments"

				script {
					echo "Build Image"
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.selector("bc", "springbootsra").startBuild("--from-dir=./ocp","--follow", "--wait=true")
						}
					}
				}
            }
          }
		  
		stage('Create DEV') {
			when {
				expression {
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							return !openshift.selector("dc", "springbootsra").exists()
						}
					}
				}
            }
			
            steps {
				script {
					echo "Create DEV"
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							def app = openshift.newApp("springbootsra:latest")
							app.narrow("svc").expose();
			
							openshift.set("probe dc/springbootsra --readiness --get-url=http://:8080/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
							openshift.set("probe dc/springbootsra --liveness  --get-url=http://:8080/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")
			
							def dc = openshift.selector("dc", "springbootsra")
							//while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
							//	sleep 10
							//}
							openshift.set("triggers", "dc/springbootsra", "--manual")
						}
					}
				}
            }
		}		  
		
		stage('Deploy DEV') {
			steps {
				script {
					echo "Deploy DEV"
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.selector("dc", "springbootsra").rollout().latest();

							if(!deployment.exists()){ 
                				openshift.newApp("springbootsra", "--as-deployment-config").narrow("svc").expose() 
              				} 
    
              				timeout(5) { 
                				openshift.selector("dc", "springbootsra").related("pods").untilEach(1) { 
                  					return (it.object().status.phase == "Running") 
                				} 
              				}
						}
					}
				}
			}
		}
		
		
	}
}