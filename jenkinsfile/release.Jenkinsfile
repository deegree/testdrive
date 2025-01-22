pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    tools {
        maven 'maven-3.9'
        jdk 'temurin-jdk17'
    }
    environment {
        MAVEN_OPTS='-Djava.awt.headless=true'
    }
    parameters {
          string name: 'REL_VERSION', defaultValue: "1.0.x", description: 'Next release version'
          string name: 'DEV_VERSION', defaultValue: "1.0.x-SNAPSHOT", description: 'Next snapshot version'
          booleanParam name: 'PERFORM_RELEASE', defaultValue: false, description: 'Perform release build (on main branch only)'
    }
    stages {
        stage ('Initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                '''
                sh 'mvn -version'
                sh 'java -version'
                sh 'git --version'
            }
        }
        stage ('Build') {
            steps {
               echo 'Building'
               sh 'mvn -B -C clean test-compile'
            }
        }
        stage ('Test') {
            steps {
                echo 'Testing'
                sh 'mvn -B -C -fae install'
            }
            post {
                always {
                    junit '**/target/*-reports/*.xml'
                }
            }
        }
        stage ('Quality Checks') {
            when {
                branch 'main'
            }
            steps {
                echo 'Quality checking'
                sh 'mvn -B -C -fae com.github.spotbugs:spotbugs-maven-plugin:spotbugs javadoc:javadoc'
            }
            post {
                always {
                    recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc(), spotBugs()]
                }
            }
        }
        stage ('Release') {
            when {
                allOf {
                    triggeredBy cause: "UserIdCause", detail: "tmc"
                    expression { return params.PERFORM_RELEASE }
                }
            }
            steps {
                echo 'Prepare release version ${REL_VERSION}'
                sshagent(credentials: ['jenkins-testdrive-ssh-key']) {
                    withMaven(mavenSettingsConfig: 'mvn-server-settings', options: [junitPublisher(healthScaleFactor: 1.0)], publisherStrategy: 'EXPLICIT') {
                        withCredentials([usernamePassword(credentialsId:'nexus-deploy', passwordVariable: 'PASSWORD_VAR', usernameVariable: 'USERNAME_VAR')]) {
//                        sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY"'
//                        sh 'git -c core.sshCommand="ssh -i $SSH_KEY" submodule update --init'
//                        catchError {
//                            sh 'eval "ssh-agent -s"'
//                            sh 'ssh -T -ai $SSH_KEY git@github.com'
//                        }
                        sh 'mvn -B release:clean release:prepare -Dresume=false -DreleaseVersion=${REL_VERSION} -DdevelopmentVersion=${DEV_VERSION}'
                        sh 'mvn -B release:perform -DdeployAtEnd=true -Dgoals=deploy -Drepo.username=${USERNAME_VAR} -Drepo.password=${PASSWORD_VAR}'
                      }
                    }
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }
    }
    post {
        always {
            cleanWs notFailBuild: true
        }
    }
}
