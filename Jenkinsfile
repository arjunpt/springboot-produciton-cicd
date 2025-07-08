pipeline {
  agent { label 'build' }
  parameters {
    string(name: 'environment', defaultValue: 'dev', description: 'Enter environment')
    string(name: 'VERSION', defaultValue: '1.1', description: 'Enter version')
  }
  environment { 
    registry = "arjunpt/democicd" 
    registryCredential = 'dockerhub' 
    imageTag = "${params.environment}-${params.VERSION}"
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', credentialsId: 'GitlabCred', url: 'https://gitlab.com/learndevopseasy/devsecops/springboot-build-pipeline.git'
      }
    }
    stage('Stage I: Build') {
      steps {
        echo "Building Jar Component ..."
        sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn clean package"
      }
    }
    stage('Stage II: Code Coverage') {
      steps {
        echo "Running Code Coverage ..."
        sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn jacoco:report"
      }
    }
    stage('Stage III: SCA') {
      steps {
        echo "Running OWASP Dependency-Check ..."
        sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn org.owasp:dependency-check-maven:check"
      }
    }
    stage('Stage IV: SAST') {
      steps {
        echo "Running SonarQube Scanner ..."
        withSonarQubeEnv('mysonarqube') {
          sh 'mvn sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json -Dsonar.dependencyCheck.htmlReportPath=target/dependency-check-report.html -Dsonar.projectName=wezvatech'
        }
      }
    }
    stage('Stage V: Quality Gates') {
      steps {
        echo "Waiting for Quality Gate result ..."
        script {
          timeout(time: 1, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }
    stage('Stage VI: Build Image') {
      steps {
        script {
          echo "Building Docker Image with tag: ${env.imageTag}"
          docker.withRegistry('', registryCredential) {
            def image = docker.build("${registry}:${env.imageTag}")
            image.push()
          }
        }
      }
    }
    stage('Stage VII: Scan Image') {
      steps {
        echo "Scanning Docker Image for Vulnerabilities"
        sh "trivy image --scanners vuln --offline-scan ${registry}:${env.imageTag} > trivyresults.txt"
      }
    }
    stage('Stage VIII: Smoke Test') {
      steps {
        echo "Smoke Testing the Docker Image"
        sh "docker run -d --name smokerun -p 8080:8080 ${registry}:${env.imageTag}"
        sh "sleep 90; ./check.sh"
        sh "docker rm --force smokerun"
      }
    }
    stage('Stage IX: Trigger CD Pipeline') {
      steps {
        script {
          def tag = "${params.environment}-${params.VERSION}"
          echo "Triggering CD job after successful CI build"
          build job: 'springboot-cd-pipeline', 
            parameters: [
              string(name: 'IMAGETAG', value: tag),
              string(name: 'environment', value: params.environment)
            ]
        }
      }
    }
  }
}
