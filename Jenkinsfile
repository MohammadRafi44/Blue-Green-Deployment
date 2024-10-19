pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "mohammadrafi44/bankapp"
        TAG = "${params.DOCKER_TAG}"  // The image tag now comes from the parameter
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME= tool 'sonar-scanner'
    }
    tools {
        maven 'maven3'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/MohammadRafi44/Blue-Green-Deployment.git'
                //git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/jaiswaladi246/3-Tier-NodeJS-MySql-Docker.git'
            }
        }
        
        stage('compile'){
            steps {
                sh 'mvn compile'
            }
        }
        
        stage('Tests') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('trviy-fs-scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Multitier \
                        -Dsonar.projectName=Multitier \
                        -Dsonar.java.binaries=target"
                    }
                }
            }
        }

        stage('Quality-Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false

                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('publish artifact to nexus') {
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh "mvn deploy -DskipTests=true"
                    }
                }
            }
        }

        stage('Docker Build & Tag Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                    sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }

        stage('Trivy image scan'){
            steps {
                sh "trivy image --format table -o fs.html ${IMAGE_NAME}:${TAG}"
            }
        }

        stage('Docker Push Image'){
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                    sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EAE2FE1C26B8260A1FE5AB7EFBB2CFC1.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready
                    }
                }
            }
        }

        stage('Deploy SVC-APP') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EAE2FE1C26B8260A1FE5AB7EFBB2CFC1.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                   }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EAE2FE1C26B8260A1FE5AB7EFBB2CFC1.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EAE2FE1C26B8260A1FE5AB7EFBB2CFC1.gr7.ap-south-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EAE2FE1C26B8260A1FE5AB7EFBB2CFC1.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
