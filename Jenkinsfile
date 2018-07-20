node('jenkins-slave.usbank.com') {
	stage('Poll') {
		checkout scm
	}
	stage('Build & Unit Test') {
		withMaven(maven: 'M3') {
			sh 'mvn clean verify -DskipITs=true';
		}
		junit '**/target/surefire-reports/TEST-*.xml'
		archive 'target/*.jar'
	}
	stage('Static Code Analysis') {
		withMaven(maven: 'M3') {
			sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project -Dsonar.projectKey=example-project -Dsonar.projectVersion=$BUILD_NUMBER';
		}
	}
	stage('Integration Test') {
		withMaven(maven: 'M3') {
			sh 'mvn clean verify -Dsurefire.skip=true';
		}
		junit '**/target/failsafe-reports/TEST-*.xml'
		archive 'target/*.jar'	
	}
	stage('Publish') {
		def server = Artifactory.server 'Artifactory Server'
		def uploadSpec = """{
			"files": {
				{
					"pattern": "target/hello-0.0.1.war",
					"target": "helloworld-greeting-project/${BUILD_NUMBER}/",
					"props": "Integration-Tested=Yes;Performance-Tested=No"
				}
			}
		}"""
		server.upload(uploadSpec)
	}
}
