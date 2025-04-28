pipeline {
    agent any

    /* ───── toolchains ───── */
    tools {
      jdk   'Java 21'
      maven 'Maven 3.8.1'
    }

    /* ───── global env ───── */
    environment {
      JAVA_HOME    = tool 'Java 21'
      M2_HOME      = tool 'Maven 3.8.1'
      SCANNER_HOME = tool 'sonar-scanner'
      PATH         = "${JAVA_HOME}/bin:${M2_HOME}/bin:${SCANNER_HOME}/bin:${env.PATH}"
      TRIVY_RESULTS_FILE='trivy_results.txt'
      DAST_SCAN_RESULTS_FILE = 'dast_scan_results.txt'
      KUBECONFIG = '/var/lib/jenkins/.kube/config'
      KUBERNETES_CREDENTIALS='k8s-cred'
      GIT_CREDENTIALS='git-cred'
      DOCKER_IMAGE = 'your-docker-image-name'  // Define your Docker image name here
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Source Code Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: 'https://github.com/sanika6969/DotNet-Application.git' //change the github repo to dotnet
            }
        }

        stage('Application Filesystem Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .' //check trivy -h { for source code file system scan }
                // Download the HTML template (optional)
                sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl'
            }
        }

        stage('Application Image Build and Push') {
            steps {
                dir('devsecops/dotnet-demoapp') {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                            sh '''
                                echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                                docker build -t ${DOCKER_IMAGE} .
                                docker push ${DOCKER_IMAGE}
                            '''
                        }
                    }
                }
            }
        }

        stage('Image Scan') {
            steps {
                sh """
                    sudo docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy:latest image --format table ${DOCKER_IMAGE}:latest > ${TRIVY_RESULTS_FILE}
                """
            }
        }

        stage('Push Application Artifacts to Dockerhub') {
            steps {
                script {
                    withDockerRegistry(credentialsId:"${DOCKER_CREDENTIALS}", toolName: 'docker') {
                        sh 'docker push sanika2003/dotnet-demoapp:latest'
                    }
                }
            }
        }

        stage('Containerize the application') {
            steps {
                withKubeConfig(credentialsId: 'k8s-cred') {
                    // sh 'kubectl apply -f k8s-manifest/'  //not required
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8s-cred') {
                    sh '''
                        kubectl get pods -n webapps
                        kubectl get svc -n webapps
                        echo "App running on 10.25.157.151:5000"
                    ''' // just add the application URL as echo command.
                }
            }
        }
    }

    post {
        always {
            script {
                // Archive artifacts
                archiveArtifacts artifacts: 'check.html'

                // Publish HTML report
                publishHTML target: [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'trivy-fs-report.html',
                    reportName: 'Trivy Filesystem Scan',
                    reportTitles: 'Trivy Filesystem Scan'
                ]
            }
        }

        success {
            script {
                def email = sh(
                    script: "git --no-pager show -s --format='%ae'",
                    returnStdout: true
                ).trim()
                mail to:      email,
                     subject: "✅ Deployment Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: """
Hello,

Your commit triggered a successful deployment for job '${env.JOB_NAME}' (build #${env.BUILD_NUMBER}).

See details: ${env.BUILD_URL}

Best,
Jenkins CI/CD
                     """
            }
        }
    }
}
