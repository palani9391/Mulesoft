#!/usr/bin/env groovy
@Library('SharedPipelines')_
pipeline {
	agent { label 'linux' }
	environment {
		APPLICATION_NAME = 'MuleSoft'
		COMPONENT_NAME = 'ateb-s-api'
		JENKINS_CRED_SCOPE = 'NonProd'
		ARTEFACT_EXTENSION = '.jar'
		TOOL_MAVEN = 'Maven_3.9.0-Linux'
		TOOL_JDK = 'Corretto 17-Linux'
	}
	tools {
		maven "$TOOL_MAVEN"
		jdk "$TOOL_JDK"
	}	
	options {
        timeout(time: 1, unit: 'HOURS')
		skipDefaultCheckout()
      	buildDiscarder(logRotator(numToKeepStr: '5'))	
		disableConcurrentBuilds()
    }
	stages {
		stage('Prepare') {
			steps {
				script {
					//check whether valid branch name
					if (!(branchNamingCheck())) {
						error ('branch does not match required naming convention')
						//todo can we email dev instructing them to delete? or even go ahead and delete?
					}
					if (checkJobTrigger()){
						echo "job was started by a user so we assume a deployment is required.."
						env.BUILD_TYPE = 'CICD'
					} else {
						echo 'job was started by an automatic trigger so we assume deployment is not required..'
						env.BUILD_TYPE = 'CI'
					}
					if (branchNamingCheck(teamsuffixes:'MUL')) {
						echo 'set target env as BAUDIT'
						env.TARGET_ENVIRONMENT = 'BAUDIT'
                    } else if (branchNamingCheck(teamsuffixes:'PR')) {
						echo 'set target env as BAUDIT'
						env.TARGET_ENVIRONMENT = 'BAUDIT'
					} else {
						echo 'set target env as BAUDIT by default'
						env.TARGET_ENVIRONMENT = 'BAUDIT'
					}
					checkoutSource(releasearea:'s3',deployframework:'extract')
				}
			}
        }
		stage ('Build') {
			steps {
				script {runMavenPackage(releasearea:'s3',mavencentral:'false',artefactext:'.jar')}
			}
		}
		stage ('Unit Test') {
			steps {
				script {runUnitTests(buildtool:'mule')}
			}
		}
		stage ('Sonarqube') {
			steps {
				script {runSonarQubeAnalysis()}
			}
		}
		stage ('Quality Gate') {
			steps {
				script {checkSQQualityGate()}
			}
		}
		stage ('RepackageForEnv') {
			when {
				beforeAgent true
				anyOf {branch 'develop'; branch 'release/*'; branch 'hotfix/*'}
				expression {
					return (env.BUILD_TYPE == 'CICD' && (env.TARGET_ENVIRONMENT));
				}
			}
			agent { label 'mule-detokenise' }
			steps {
				script { repackageForMuleApi() }
			}
		}
		stage ('IaC Build Env') {
			when {
				beforeAgent true
				anyOf {branch 'develop'; branch 'release/*'; branch 'hotfix/*'}
				expression {
					return (env.BUILD_TYPE == 'CICD' && (env.TARGET_ENVIRONMENT));
			}
		}
		agent { label 'mule-detokenise' }
			steps {
				script { runMuleApiBuildEnv() }
			}
		}
		stage ('Deploy') {
			when {
				beforeAgent true
				anyOf {branch 'develop'; branch 'release/*'; branch 'hotfix/*'}
				expression {
					return (env.BUILD_TYPE == 'CICD' && (env.TARGET_ENVIRONMENT));
				}
			}
			agent { label 'deploy-mule-non-prod' }
			steps {
				script { deployMuleApis() }
			}
		}
		stage ('Tag') {
			when {
				beforeAgent true
				anyOf {branch 'develop'; branch 'release/*'; branch 'hotfix/*'}
				expression {
					return (env.BUILD_TYPE == 'CICD');
				}
			}
			steps {
				script { tagSourceCode() }
			}
		}
	}
	post {
		always {
			script {
              	sendEmailNotifications (buildStatus:currentBuild.result, reqPlutora:'true')
				echo 'cleaning workspace!!!'
				deleteDir()
			}
		}
	}
}
