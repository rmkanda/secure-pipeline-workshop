pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets scanner') {
          steps {
            container('trufflehog') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "trufflehog --exclude_paths secrets-exclude.txt ${GIT_URL}"
              }
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('Dependency Checker') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            }
          }
        }
        stage('Spot Bugs - Security') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw compile spotbugs:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/spotbugsXml.xml', fingerprint: true, onlyIfSuccessful: false
              recordIssues enabledForFailure: true, tool: spotBugs()
            }
          }
        }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
                    '''
              }
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              sh './mvnw org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }
          }
          post {
            success {
              // dependencyTrackPublisher projectName: 'sample-spring-app', projectVersion: '0.0.1', artifact: 'target/bom.xml', autoCreateProjects: true, synchronous: true
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
      }
    }
    stage('Artifacts Analysis') {
      parallel {
        stage('Container Scan') {
          steps {
            container('docker-tools') {
              sh "grype ${APP_NAME} || exit 0"
            }
          }
        }
        stage('Dockle') {
          steps {
            container('docker-tools') {
              sh "dockle ${APP_NAME}"
            }
          }
        }
        stage('Kubesec') {
          steps {
            container('docker-tools') {
              sh 'kubesec scan k8s.yaml || exit 0'
            }
          }
        }
      }
    }
    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }
    stage('DAST') {
      steps {
        container('docker-tools') {
          sh "docker pull owasp/zap2docker-weekly"
          sh "docker run -t owasp/zap2docker-stable zap-api-scan.py -t http://172.17.0.4:30001/v3/api-docs -f openapi"
        }
      }
    }
    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}