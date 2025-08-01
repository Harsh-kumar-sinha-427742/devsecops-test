pipeline {
    agent any

    environment {
        //DEPENDENCY_CHECK = '/opt/dependency-check/dependency-check/bin/dependency-check.sh'
        SONAR_SCANNER = tool name: 'sonar-scanner'
        SONAR_URL =  'http://16.16.25.45:9000'   //ip of sonarqube
        ZAP_REPORT_HTML = 'zap_report.html'
        ZAP_REPORT_XML  = 'zap_report.xml'
        ZAP_REPORT_JSON = 'zap_report.json'
        TARGET_URL      = "http://${IP_HOSTED}:3000" // Replace with actual target(deployed juice-shop
        IMAGE_NAME = 'kumar0ndocker/juice-shop'
        TAG = 'v2'
        IP_HOSTED = '13.60.182.131' //juice shop hosted ip
        WEB_HOST = "ubuntu@${IP_HOSTED}"
        IP = '16.170.162.214'                          //jenkins-server ip
        EC2_HOST          = "ubuntu@${IP}"
        WEB_APP_PORT      = '3000'
        EC2_KEY_ID        = 'ec2-ssh-key'
        ZAP_INSTANCE_HOST = "ubuntu@13.60.251.28"       //DAST -SCAN
    }

    stages {
        
        // running
        stage('Deploy App to AWS EC2') {
            steps {
                echo '🚀 Deploying Juice Shop to EC2...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                     sh """#!/bin/bash
                        ssh -o StrictHostKeyChecking=no $WEB_HOST << 'ENDSSH'
                            echo "🔍 Checking for any process on port $WEB_APP_PORT..."
                            PID=\$(sudo lsof -t -i:$WEB_APP_PORT)
                            if [ ! -z "\$PID" ]; then
                                echo "🛑 Killing process \$PID using port $WEB_APP_PORT"
                                sudo kill -9 \$PID || true
                            else
                                echo "✅ No process found on port $WEB_APP_PORT"
                            fi
        
                            echo "🧹 Removing old Docker container..."
                            docker rm -f juice-shop || true
        
                            echo "📦 Pulling Docker image: $IMAGE_NAME:$TAG"
                            docker pull $IMAGE_NAME:$TAG
        
                            echo "🚀 Running new Docker container..."
                            docker run -d -p $WEB_APP_PORT:$WEB_APP_PORT --name juice-shop $IMAGE_NAME:$TAG
        
                            echo "✅ Deployment complete"
                        ENDSSH
                        sleep 20
                    """
                }
            }
        }
        
        /*
        stage('Clone Repository') {
            steps {
                echo 'Cloning the GitHub Repository...'
                sh '''
                    rm -rf temp_repo
                    git clone --depth=1 https://github.com/Akashsonawane571/devsecops-test.git temp_repo
                '''
            }
        }
        // TruffleHog is installed locally
        
        stage('Secret Scan (Trufflehog)') {
            steps {
                echo 'Running Trufflehog on latest commit only...'
                sh '''
                  cd temp_repo
                  trufflehog --regex --entropy=True --max_depth=10 . > ../trufflehog_report.json || true
                  cd ..
                '''
                archiveArtifacts artifacts: 'trufflehog_report.json', onlyIfSuccessful: false
            }
        }
        
     
        stage('Remote Dependency Check (Offline)') {
            steps {
                echo '🔍 Running Dependency Check offline with no-update on EC2...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                    sh """
                        echo '📤 Copying source repo to EC2...'
                        chmod -R u+rwx temp_repo
                        ssh -o StrictHostKeyChecking=no $EC2_HOST 'rm -rf ~/temp_repo'
                        scp -o StrictHostKeyChecking=no -r temp_repo $EC2_HOST:~/temp_repo
        
                        echo '📦 Running Dependency-Check on EC2...'
                        ssh -o StrictHostKeyChecking=no $EC2_HOST '
                            mkdir -p ~/odc-report &&
                            /opt/dependency-check/dependency-check/bin/dependency-check.sh \
                                --project "Remote-Scan" \
                                -s ~/temp_repo \
                                -o ~/odc-report \
                                -f ALL \
                                --data ~/odc-data \
                                --noupdate || true
                        '
        
                        echo '📥 Copying report back to Jenkins workspace...'
                        scp -o StrictHostKeyChecking=no $EC2_HOST:~/odc-report/dependency-check-report.* .
                    """
                }
                archiveArtifacts artifacts: 'dependency-check-report.*', onlyIfSuccessful: false
            }
        }
     
                  
        stage('SonarQube Scan') {
            steps {
                echo 'Starting SonarQube SAST Scan...'
                withSonarQubeEnv('sonarqube') {
                    withCredentials([string(credentialsId: 'newtoken', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            cd temp_repo
                            $SONAR_SCANNER/bin/sonar-scanner \
                              -Dsonar.projectKey=devsecops-test \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=$SONAR_URL \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }
        
        
   
        //running h
        stage('Build Project') {
            steps {
                echo 'Building the Java project with Maven...'
                dir('temp_repo') {
                    sh 'mvn clean install -DskipTests'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                dir('temp_repo') {
                    script {
                        sh "docker build -t $IMAGE_NAME:$TAG ."
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push $IMAGE_NAME:$TAG
                            docker logout
                        '''
                    }
                }
            }
        }
        
        
       // running
        stage('Run ZAP DAST Scan (Baseline)') {
            steps {
                echo 'Running ZAP Baseline DAST Scan...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $ZAP_INSTANCE_HOST '
                            mkdir -p ~/zap-work && chmod -R 777 ~/zap-work &&
                            docker run --rm \
                                -v ~/zap-work:/zap/wrk/:rw \
                                zaproxy/zap-stable \
                                zap-baseline.py -t $TARGET_URL \
                                -r zap_report.html -x zap_report.xml -J zap_report.json || true
                        '
                        scp -o StrictHostKeyChecking=no $ZAP_INSTANCE_HOST:~/zap-work/zap_report.* .
                    """
                    archiveArtifacts artifacts: "${ZAP_REPORT_HTML}, ${ZAP_REPORT_XML}, ${ZAP_REPORT_JSON}", onlyIfSuccessful: false
                }
        
                // Copy the reports back (optional, if needed on Jenkins instance)
                // You can also add scp commands if needed
            }
        }
        stage('Run Nikto Scan') {
            steps {
                echo 'Running Nikto scan on remote ZAP instance...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $ZAP_INSTANCE_HOST '
                            mkdir -p ~/zap-work &&
                            nikto -h $TARGET_URL -output ~/zap-work/nikto_report.html -Format html \\
                            -Display V \\
                            -Plugins ALL \\
                            -Tuning 1234567890 \\
                            -Cgidirs all \\
                            -Useragent "NiktoScanner/1.1" \\
                            -no404 
                        '
                        echo "📥 Copying Nikto report to Jenkins workspace..."
                        scp -o StrictHostKeyChecking=no $ZAP_INSTANCE_HOST:~/zap-work/nikto_report.html .
                    """
                    archiveArtifacts artifacts: 'nikto_report.html', onlyIfSuccessful: false
                }
            }
        }  
        */
    }

    post {
        always {
            script {
                echo 'Cleaning up temporary files...'
                sh '''
                    rm -rf temp_repo dependency-check-report trufflehog_report.txt juice-shop \
                           $ZAP_REPORT_HTML $ZAP_REPORT_XML $ZAP_REPORT_JSON || true
                '''
            }
        }
    }
}
