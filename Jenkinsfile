pipeline {
    agent any

    environment {
        SONAR_HOME = tool "Sonar"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Frontend Docker tag')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Backend Docker tag')
    }

    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (!params.FRONTEND_DOCKER_TAG || !params.BACKEND_DOCKER_TAG) {
                        error("Both FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }

        stage("Git: Code Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/ram-nivas234/Wanderlust-Mega-Project.git'
            }
        }

        stage("Trivy: Filesystem Scan") {
            steps {
                sh 'trivy fs --exit-code 0 --severity LOW,MEDIUM,HIGH --no-progress .'
            }
        }

        stage("OWASP: Dependency Check") {
            steps {
                sh 'dependency-check.sh --scan . --format XML --project Wanderlust'
            }
        }

        stage("SonarQube: Code Analysis") {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh "${SONAR_HOME}/bin/sonar-scanner -Dsonar.projectKey=wanderlust -Dsonar.sources=. -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$SONAR_AUTH_TOKEN"
                }
            }
        }

        stage("SonarQube: Code Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Set Environment Variables") {
            parallel {
                stage("Backend Env Setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }

                stage("Frontend Env Setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }

        stage("Docker: Build Images") {
            steps {
                script {
                    dir('backend') {
                        sh "docker build -t trainwithshubham/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} ."
                    }
                    dir('frontend') {
                        sh "docker build -t trainwithshubham/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} ."
                    }
                }
            }
        }

        stage("Docker: Push Images") {
            steps {
                script {
                    sh "docker push trainwithshubham/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}"
                    sh "docker push trainwithshubham/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}"
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: 'Wanderlust-CD', parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: params.FRONTEND_DOCKER_TAG),
                string(name: 'BACKEND_DOCKER_TAG', value: params.BACKEND_DOCKER_TAG)
            ]
        }
    }
}
