pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REPO = '605134473022.dkr.ecr.ap-northeast-2.amazonaws.com/jenkins-images'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build JAR') { 
            steps {
                sh 'chmod +x gradlew' // gradlew에 실행 권한을 부여
                sh './gradlew clean build'  // Gradle 빌드 수행
            }
        }

        // OWASP Dependency Check
     //    stage('OWASP Dependency-Check Vulnerabilities') {
     //        steps {
     //            dir("src"){
     //                dependencyCheck additionalArguments: ''' 
     //                -o './'
     //                -s './'
     //                -f 'ALL'
     //                --nvdApiKey '8f6d2492-c22e-4824-bfe8-95c7664dec4a'
     //                --prettyPrint''', odcInstallation: 'owasp'
    
     //                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
     //            }
     //        }
    	// }

        // SonarQube 분석
        stage('SonarQube Scanner') {
            steps {
                withSonarQubeEnv('jg-sonarqube') {
                    sh "./gradlew sonar"
                }
            }
        }

        // image build
        stage('Build Image') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REPO}/", '9b45eaf4-a184-44eb-ba8c-8e20a854de1b') {
                        myapp = docker.build('jenkins-images')
                    }
                }
            }
        }

        // image scan ( Trivy )
        stage('Scan Image with Trivy') {
            steps {
                script {
                    try {
                        // Trivy로 이미지 스캔하고 HTML 리포트 생성
                        sh 'trivy image --format template --template "@/root/html.tpl" --output trivy-report.html "${ECR_REPO}:${IMAGE_TAG}"'
                        echo "Trivy scan completed"
                    } catch (Exception e) {
                        echo "Trivy scan failed: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        throw e 
                    }
                }
            }
        }

        stage('Publish Trivy Report') {
            steps {
                script {
                    // HTML 리포트가 존재하는지 확인하고 리포트를 출력
                    if (fileExists('trivy-report.html')) {
                        echo "Trivy report found, publishing HTML report"
                        publishHTML(target: [
                            allowMissing: false,
                            alwaysLinkToLastBuild: false,
                            keepAll: false,
                            reportDir: '.',
                            reportFiles: 'trivy-report.html',
                            reportName: 'Trivy Vulnerability Report'
                        ])
                    } else {
                        echo "Trivy report not found, skipping HTML report publishing"
                    }
                }
            }
        }

        // iamge push
        stage('Push Image to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REPO}/", '9b45eaf4-a184-44eb-ba8c-8e20a854de1b') {
                        myapp.push("${IMAGE_TAG}")
                    }
                }
            }
        }
    }
}
