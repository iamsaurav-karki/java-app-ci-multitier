pipeline {
  agent any
  
  parameters {
      string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'Docker tag')
  }
  
  tools {
    maven 'maven3'
  }
  environment {
    SCANNER_HOME = tool 'sonar-scanner'
    SNYK_TOKEN = credentials('snyk-token')
  }
  stages {
    stage('Clean Workspace') {
      steps {   
        cleanWs()
      }
    }
    stage('Git Checkout') {
      steps {   
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/iamsaurav-karki/Multi-Tier-BankApp-CI.git'
      }
    }

    stage('Snyk Source Code Scan') {
      steps {
        script {
            withEnv(["SNYK_TOKEN=${env.SNYK_TOKEN}"]) {
                sh 'snyk test --severity-threshold=high'
            }
        }
      }
    }

    stage('Compile') {
      steps {
        sh "mvn compile"
      }
    }

    stage('Unit Test Cases') {
      steps {
        sh "mvn test -DskipTests=true"
      }
    }

    stage('Trivy FS scan') {
      steps {
        sh "trivy fs --format table -o fs.html ."
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv('sonar-server') {
          sh ''' 
          ${SCANNER_HOME}/bin/sonar-scanner \
          -Dsonar.projectKey=Multi-Tier-BankApp-CI \
          -Dsonar.projectName=Multi-Tier-BankApp-CI \
          -Dsonar.java.binaries=target 
          '''
        }
      }
    }

    stage('Build and Publish to Nexus') {
      steps {
        withMaven(globalMavenSettingsConfig: 'maven-settings',jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
          sh "mvn deploy -DskipTests=true"
        }
      }
    }
    
    stage('Docker Build and Tag') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
            sh "docker build -t sauravkarki/bankappnew:${params.DOCKER_TAG ?: 'latest'} ."
          }
        }
      }
    }
    
    stage('Snyk Docker Image Scan') {
      steps {
        script {
            withEnv(["SNYK_TOKEN=${env.SNYK_TOKEN}"]) {
                sh "snyk test --docker sauravkarki/bankappnew:${params.DOCKER_TAG} --file=Dockerfile --severity-threshold=high"
            }
        }
      }
    }

    stage('Docker Image scan') {
      steps {
        sh "trivy image --format table -o dimage.html sauravkarki/bankappnew:${params.DOCKER_TAG}"
      }
    }
    
    stage('Publish To Docker Hub') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
            sh "docker push sauravkarki/bankappnew:${params.DOCKER_TAG}"
          }
        }
      }
    }
    
    stage('Update Image Tag in CD Repo') {
      steps {
        script {
            withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                sh '''
                if [ -d "Multi-Tier-BankApp-CD" ]; then
                    echo "Removing existing Multi-Tier-BankApp-CD directory..."
                    rm -rf Multi-Tier-BankApp-CD
                fi

                git clone https://github.com/iamsaurav-karki/Multi-Tier-BankApp-CD.git

                cd Multi-Tier-BankApp-CD

                if [ -z "${DOCKER_TAG}" ]; then
                    echo "ERROR: DOCKER_TAG is not set!"
                    exit 1
                fi

                sed -i "s|image: sauravkarki/bankappnew:.*|image: sauravkarki/bankappnew:${DOCKER_TAG}|" bankapp/bankapp-ds.yml

                echo "Updated YAML file:"
                cat bankapp/bankapp-ds.yml

                git config user.email "sauravkarki102@gmail.com"
                git config user.name "iamsaurav-karki"

                git add bankapp/bankapp-ds.yml

                git commit -am "Updated the YAML manifests with the recent image tag" || echo "No changes to commit"

                git push origin main
                '''
            }
        }
      }
    }
  }
}
