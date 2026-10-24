
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-2'
        ARTIFACT_BUCKET = 'aa-maven-tomcat-artifacts-2026'
        TOMCAT_INSTANCE_ID = 'i-0ade39f2b00cb338d'
        WAR_NAME = 'maven-tomcat-demo.war'
        S3_KEY = 'maven-tomcat-demo/maven-tomcat-demo.war'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/udaykn13811-hub/maven-tomcat-demo.git'
            }
        }

        stage('Build WAR with Maven') {
            steps {
                sh 'mvn -B clean package'
                sh 'test -f target/maven-tomcat-demo.war'
            }
        }

        stage('Archive WAR') {
            steps {
                archiveArtifacts artifacts: 'target/*.war',
                                 fingerprint: true
            }
        }

        stage('Upload WAR to S3') {
            steps {
                sh '''
                    aws s3 cp target/${WAR_NAME} \
                      s3://${ARTIFACT_BUCKET}/${S3_KEY}
                '''
            }
        }

        stage('Deploy to Tomcat through SSM') {
            steps {
                sh '''
                    set -eu

                    COMMAND_ID=$(aws ssm send-command \
                      --instance-ids "$TOMCAT_INSTANCE_ID" \
                      --document-name "AWS-RunShellScript" \
                      --parameters 'commands=["aws s3 cp s3://aa-maven-tomcat-artifacts-2026/maven-tomcat-demo/maven-tomcat-demo.war /tmp/maven-tomcat-demo.war","rm -rf /opt/tomcat/webapps/maven-tomcat-demo","install -o tomcat -g tomcat -m 0644 /tmp/maven-tomcat-demo.war /opt/tomcat/webapps/maven-tomcat-demo.war"]' \
                      --query 'Command.CommandId' \
                      --output text)

                    echo "SSM Command ID: $COMMAND_ID"

                    for i in $(seq 1 30); do
                        STATUS=$(aws ssm get-command-invocation \
                          --command-id "$COMMAND_ID" \
                          --instance-id "$TOMCAT_INSTANCE_ID" \
                          --query 'Status' \
                          --output text 2>/dev/null || echo Pending)

                        echo "Deployment status: $STATUS"

                        case "$STATUS" in
                          Success)
                            exit 0
                            ;;
                          Failed|Cancelled|TimedOut)
                            aws ssm get-command-invocation \
                              --command-id "$COMMAND_ID" \
                              --instance-id "$TOMCAT_INSTANCE_ID" \
                              --query '{Status:Status,Output:StandardOutputContent,Error:StandardErrorContent}' \
                              --output json || true
                            exit 1
                            ;;
                        esac

                        sleep 10
                    done

                    echo "SSM deployment did not finish within the polling window."
                    exit 1
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking Tomcat application..."

                    for i in $(seq 1 12); do
                        if curl --fail --silent --show-error \
                          --connect-timeout 5 \
                          "http://172.31.1.247:8080/maven-tomcat-demo/" \
                          -o /tmp/aa-app-response.html; then

                            grep -q "Maven + Jenkins + Tomcat Integration" \
                              /tmp/aa-app-response.html

                            echo "Application verification successful."
                            exit 0
                        fi

                        sleep 10
                    done

                    echo "Application did not become healthy."
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Maven WAR built and deployed to Tomcat.'
        }
        failure {
            echo 'FAILED: Check the Jenkins console output.'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
