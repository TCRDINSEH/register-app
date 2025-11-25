pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"

        DOCKER_USER = "tcrd4323"
        DOCKER_CRED_ID = "dockercred"      // Jenkins Credentials ID

        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"

        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {

        stage("Cleanup Workspace") {
            steps { cleanWs() }
        }

        stage("Checkout SCM") {
            steps {
                git branch: 'main',
                url: 'https://github.com/TCRDINSEH/register-app.git'
            }
        }

        stage("Build Application") {
            steps { sh "mvn clean package -DskipTests" }
        }

        stage("Run Tests") {
            steps { sh "mvn test" }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CRED_ID) {
                        def dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        dockerImage.push()
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image ${IMAGE_NAME}:latest \
                --no-progress \
                --scanners vuln \
                --exit-code 0 \
                --severity HIGH,CRITICAL \
                --format table
                """
            }
        }

        stage("Cleanup Docker") {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }

    //     stage("Trigger CD Pipeline") {
    //         steps {
    //             sh """
    //             curl -X POST -u clouduser:${JENKINS_API_TOKEN} \
    //             "http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token&IMAGE_TAG=${IMAGE_TAG}"
    //             """
    //         }
    //     }
     }

    post {
        success {
            emailext(
                subject: "${JOB_NAME} - Build #${BUILD_NUMBER} SUCCESS",
                to: "dineshrgopal89@gmail.com",
                body: '${SCRIPT, template="groovy-html.template"}'
            )
        }
        failure {
            emailext(
                subject: "${JOB_NAME} - Build #${BUILD_NUMBER} FAILED",
                to: "dineshrgopal89@gmail.com",
                body: '${SCRIPT, template="groovy-html.template"}'
            )
        }
    }
}
