# Build and Deploy applications


## Prerequisites

- Jenkins installed and configured
- AWS CLI configured with appropriate credentials
- Docker installed
- Helm installed
- Kubernetes cluster configured (EKS in this case)

## Jenkins Pipeline

### Backend Service

The Jenkins pipeline for the backend service is defined in the `backend/Jenkinsfile` and includes the following stages:

1. **Read the Version**: Reads the application version from `package.json`.
2. **Install Dependencies**: Installs the necessary dependencies using `npm` for backend service.
3. **Build and Push Docker Image**: Builds the Docker image and pushes it to AWS ECR.
4. **Deploy Application**: Deploys the application using Helm.

### Frontend Service

The Jenkins pipeline for the frontend service is defined in the `frontend/Jenkinsfile` and includes similar stages as the backend service.

### Jenkinsfile for Backend
```
pipeline {
    agent {
        label 'AGENT-1'
    }
    environment {
        appVersion = '' 
        region = 'us-east-1'
        account_id = '528757792370'
        project = 'expense'
        environment = 'dev'
        component = 'backend'
    }
    stages {
        stage('Read the Version') {
            steps {
                script {
                    def packageJson = readJSON file: 'backend/package.json'
                    appVersion = packageJson.version
                    echo "App Version: ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('backend') {
                    sh 'npm install'
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    dir('backend') {
                        sh """
                            aws ecr get-login-password --region $region | docker login --username AWS --password-stdin ${account_id}.dkr.ecr.$region.amazonaws.com
                            docker build -t ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion} .
                            docker images
                            docker push ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Deploy Application') {
            steps {
                withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
                    sh """
                        aws eks update-kubeconfig --region ${region} --name ${project}-${environment}
                        cd backend/helm
                        helm upgrade --install ${component} . --set image.repository=${account_id}.dkr.ecr.${region}.amazonaws.com/${project}/${environment}/${component},image.tag=${appVersion} -f ../backend-values.yml
                    """
                }
            }
        }
    }
    post {
        always {
            deleteDir()
        }
    }
}
```

Jenkinsfile for Frontend

```
// filepath: /c:/Users/user/daws/devops/devops-repo/35-expense-apps-jenkins/frontend/Jenkinsfile
pipeline {
    agent {
        label 'AGENT-1'
    }
    environment {
        appVersion = '' // this will become global, we can use across pipeline
        region = 'us-east-1'
        account_id = '528757792370'
        project = 'expense'
        environment = 'dev'
        component = 'frontend'
    }
    stages {
        stage('Read the Version') {
            steps {
                script {
                    def packageJson = readJSON file: 'frontend/package.json'
                    appVersion = packageJson.version
                    echo "App Version: ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    dir('frontend') {
                        sh """
                            aws ecr get-login-password --region $region | docker login --username AWS --password-stdin ${account_id}.dkr.ecr.$region.amazonaws.com
                            docker build -t ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion} .
                            docker images
                            docker push ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Deploy Application') {
            steps {
                withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
                    sh """
                        aws eks update-kubeconfig --region ${region} --name ${project}-${environment}
                        cd frontend/helm
                        helm upgrade --install ${component} . --set image.repository=${account_id}.dkr.ecr.${region}.amazonaws.com/${project}/${environment}/${component},image.tag=${appVersion} -f ../frontend-values.yml
                    """
                }
            }
        }
    }
    post {
        always {
            deleteDir()
        }
    }
}
```

Helm Charts

* Each service (backend and frontend) has its own Helm chart located in the helm directory. The Helm chart includes templates for Kubernetes resources such as deployments and services.

Overriding Default Values

* You can override the default values in the Helm chart by providing a *-values.yml file. For example, backend-values.yml and frontend-values.yml are used to override default values for the backend and frontend services, respectively.

