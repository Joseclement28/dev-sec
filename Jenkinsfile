pipeline {
    agent any

    tools {
        maven "M3"
        jdk 'jdk17'
        maven 'M3'
    }

    environment {
        IMAGE = "petclinic"
        TAG = "${BUILD_NUMBER}"
        NEXUS_URL = "http://13.217.200.32:8082"
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {

        stage('1. Git Checkout') {
            steps {
                git branch: 'master',
                      url: 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('2. Maven Build + SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                      sh """
                      mvn clean verify sonar:sonar \
                      -DskipTests \
                      -Dcheckstyle.skip=true \
                      -Dsonar.projectKey=petclinic \
                      -Dsonar.login=${SONAR_TOKEN}
                      """
                    }  
                }
            }
        }

        stage('3. Maven Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('4. Nexus Deploy') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS')]) {

                    sh """
                    cat > settings.xml <<EOF
                    <settings>
                      <servers>
                        <server>
                          <id>nexus</id>
                          <username>${NEXUS_USER}</username>
                          <password>${NEXUS_PASS}</password>
                        </server>
                      </servers>
                    </settings>
        EOF

                    mvn deploy -DskipTests --settings settings.xml
                    """
                }
            }
        }

        stage('5. Docker Build') {
            steps {
                sh """
                docker build -t $IMAGE:$TAG .
                """
            }
        }

        stage('6. Trivy Scan') {
            steps {
                sh """
                trivy image --severity HIGH,CRITICAL $IMAGE:$TAG
                """
            }
        }

        stage('7. Deploy Container') {
            steps {
                sh """
                docker rm -f petclinic || true
                docker run -d --name petclinic -p 8081:8080 $IMAGE:$TAG
                """
            }
        }

        stage('8. ZAP Scan') {
            steps {
                sh """
                docker run -t zaproxy/zap-stable zap-baseline.py \
                -t http://localhost:8081
                """
            }
        }
    }
}
