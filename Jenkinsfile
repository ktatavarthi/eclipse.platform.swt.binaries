pipeline {
	options {
		skipDefaultCheckout() // Specialiced checkout is performed below
		timestamps()
		timeout(time: 90, unit: 'MINUTES')
		buildDiscarder(logRotator(numToKeepStr:'5'))
		disableConcurrentBuilds(abortPrevious: true)
	}
	agent {
		kubernetes {
			label 'swtbuild-pod'
			defaultContainer 'container'
			yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    resources:
      requests:
        memory: "512Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  - name: container
    image: akurtakov/swtbuild@sha256:2cd3dfdea1f250597c9e3797512953c03e2182344e86976c1b60117c5dd36a9e
    tty: true
    command: [ "uid_entrypoint", "cat" ]
    resources:
      requests:
        memory: "4Gi"
        cpu: "1"
      limits:
        memory: "4Gi"
        cpu: "1"
    volumeMounts:
    - name: "settings-xml"
      mountPath: "/home/jenkins/.m2/settings.xml"
      subPath: "settings.xml"
      readOnly: true
    - name: "toolchains-xml"
      mountPath: "/home/jenkins/.m2/toolchains.xml"
      subPath: "toolchains.xml"
      readOnly: true
    - name: "settings-security-xml"
      mountPath: "/home/jenkins/.m2/settings-security.xml"
      subPath: "settings-security.xml"
      readOnly: true
    - name: m2-repo
      mountPath: /home/jenkins/.m2/repository
    - name: "tools"
      mountPath: "/opt/tools"
  volumes:
  - name: settings-xml
    secret:
      secretName: m2-secret-dir
      items:
      - key: settings.xml
        path: settings.xml
  - name: toolchains-xml
    configMap:
      name: m2-dir
      items:
      - key: toolchains.xml
        path: toolchains.xml
  - name: settings-security-xml
    secret:
      secretName: m2-secret-dir
      items:
      - key: settings-security.xml
        path: settings-security.xml
  - name: m2-repo
    emptyDir: {}
  - name: tools
    persistentVolumeClaim:
      claimName: tools-claim-jiro-platform
"""
		}
	}
	environment {
		MAVEN_OPTS = "-Xmx4G"
	}
	stages {
		stage('Prepare environment') {
			steps {
				container('container') {
					dir ('eclipse.platform.swt') {
						checkout([$class: 'GitSCM', branches: [[name: '*/master']], 
							extensions: [[$class: 'CloneOption', timeout: 120]], userRemoteConfigs: [[url: 'https://github.com/eclipse-platform/eclipse.platform.swt.git']]
						])
					}
					dir ('eclipse.platform.swt.binaries') {
						checkout scm
					}
				}
			}
		}
		stage('Build') {
			steps {
				container('container') {
					wrap([$class: 'Xvnc', useXauthority: true]) {
						withEnv(["JAVA_HOME=${ tool 'openjdk-jdk17-latest' }"]) {
							dir ('eclipse.platform.swt.binaries') {
								sh '''
									/opt/tools/apache-maven/latest/bin/mvn install \
										--batch-mode --no-transfer-progress \
										-DforceContextQualifier=zzz -Dnative=gtk.linux.x86_64 \
										-Dcompare-version-with-baselines.skip=true -Dmaven.compiler.failOnWarning=true
								'''
							}
							dir ('eclipse.platform.swt') {
								sh '''
									/opt/tools/apache-maven/latest/bin/mvn clean verify \
										--batch-mode --no-transfer-progress \
										-DcheckAllWS=true -DforkCount=0 \
										-Dcompare-version-with-baselines.skip=false -Dmaven.compiler.failOnWarning=true \
										-Dmaven.test.failure.ignore=true -Dmaven.test.error.ignore=true
								'''
							}
						}
					}
				}
			}
			post {
				always {
					junit '**/*.test*/target/surefire-reports/*.xml'
					archiveArtifacts artifacts: '**/*.log,**/*.html,**/target/*.jar,**/target/*.zip'
					publishIssues issues:[scanForIssues(tool: java()), scanForIssues(tool: mavenConsole())]
				}
			}
		}
	}
}
