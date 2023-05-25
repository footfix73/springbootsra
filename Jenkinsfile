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
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.newBuild("--name=springbootsra", "--image-stream=registry.redhat.io/openjdk/openjdk-11-rhel7", "--binary=true")
						}
					}
				}
            }
		}
		
		stage('Build Image') {
			steps {
				sh "rm -rf ocp && mkdir -p ocp/deployments"
				sh "pwd && ls -la target "
				sh "cp target/k8sistio-*.jar ocp/deployments"

				script {
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.selector("bc", "demo-1-app").startBuild("--from-dir=./ocp","--follow", "--wait=true")
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
							return !openshift.selector('dc', 'demo-1-app').exists()
						}
					}
				}
            }
			
            steps {
				script {
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							def app = openshift.newApp("demo-1-app:latest")
							app.narrow("svc").expose();

							//http://localhost:8080/actuator/health
							openshift.set("probe dc/demo-1-app --readiness --get-url=http://:8080/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
							openshift.set("probe dc/demo-1-app --liveness  --get-url=http://:8080/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")

							def dc = openshift.selector("dc", "demo-1-app")
							while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
								sleep 10
							}
							openshift.set("triggers", "dc/demo-1-app", "--manual")
						}
					}
				}
            }
		}		  
		
		stage('Deploy DEV') {
			steps {
				script {
					openshift.withCluster() {
						openshift.withProject("vicentegarcia-dev") {
							openshift.selector("dc", "demo-1-app").rollout().latest();
						}
					}
				}
			}
		}
		
		
	}
}