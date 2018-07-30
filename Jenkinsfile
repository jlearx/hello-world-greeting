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
			"files": [
				{
					"pattern": "target/hello-0.0.1.war",
					"target": "generic-local/${BUILD_NUMBER}/",
					"props": "Integration-Tested=Yes;Performance-Tested=No"
				}
			]
		}"""
		server.upload(uploadSpec)
	}
	stash includes: 'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name: 'binary'
}

node('qatest.usbank.com') {
	stage('Start Tomcat') {
		sh '''cd /home/jenkins_agent/tomcat/bin
		./startup.sh''';
	}
	stage('Deploy') {
		unstash 'binary'
		sh 'cp target/hello-0.0.1.war /home/jenkins_agent/tomcat/webapps/';
	}
	stage('Performance Testing') {
		sh '''cd /opt/jmeter/bin/
		./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl''';
		step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
	}
	stage('Promote Build in Artifactory') {
		withCredentials([usernameColonPassword(credentialsId: 'f56e46b4-e75e-4a89-848b-b04dd7199747', variable: 'credentials')]) {
			sh 'curl -u${credentials} -X PUT "http://jenkins-master.usbank.com:8081/artifactory/api/storage/generic-local/${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes"';
		}
	}
}

node ('prod.usbank.com') {
	stage('Deploy to Prod') {
		def server = Artifactory.server 'Artifactory Server'
		def downloadSpec = """{
			"files": [
				{
					"pattern": "generic-local/${BUILD_NUMBER}/*.zip",
					"target": "/home/jenkins_agent/tomcat/webapps/",
					"props": "Integration-Tested=Yes;Performance-Tested=Yes"
				}
			]
		}"""
		server.download(downloadSpec)
	}
}
