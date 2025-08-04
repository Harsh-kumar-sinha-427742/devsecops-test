pipeline {
    agent any

    environment {
        //Jenkins_instance ip
        IP = '13.48.48.24'                    //jenkins-server ip
        EC2_HOST = "ubuntu@${IP}"             // for jenkins ssh

        //Sonarqube-Configuration
        SONAR_SCANNER = tool name: 'sonar-scanner'
        SONAR_URL =  'http://56.228.10.151:9000'   //ip of sonarqube

        //Application{Juice-shop}configuration
        IP_HOSTED = '13.60.250.22' //juice shop hosted ip
        IMAGE_NAME = 'kumar0ndocker/juice-shop'
        TAG = 'v4.1'        
        TARGET_URL = "http://${IP_HOSTED}:3000" // Replace with actual target(deployed juice-shop
        WEB_HOST = "ubuntu@${IP_HOSTED}"
        EC2_KEY_ID    = 'ec2-ssh-key'
        WEB_APP_PORT  = '3000'
        
        //DAST-INSTANCE-CONFIGURATION
        ZAP_INSTANCE_HOST = "ubuntu@51.20.144.204"       //DAST -SCAN
        ZAP_REPORT_HTML = 'zap_report.html'
        ZAP_REPORT_XML  = 'zap_report.xml'
        ZAP_REPORT_JSON = 'zap_report.json'
        
        //DEFECTDOJO-CONFIGURATION
        IP_DD = '51.21.246.197'      //DEFECT-DOJO IP
        DEFECTDOJO_URL = "http://${IP_DD}:8080"
        ENGAGEMENT_ID = '2'  
  
  
        
    }

    stages {
           
                
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
                  trufflehog --json --max_depth=10 . > ../trufflehog_report.json || true
                  
                  cd ..
                '''
                archiveArtifacts artifacts: 'trufflehog_report.json', onlyIfSuccessful: false
            }
        }
        
     
        stage('Software Composition Analysis') {
            steps {
                echo '🔍 Running Dependency Check offline with no-update on EC2...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                    sh """
                        echo ' Copying source repo to EC2...'
                        chmod -R u+rwx temp_repo
                        ssh -o StrictHostKeyChecking=no $EC2_HOST 'rm -rf ~/temp_repo'
                        scp -o StrictHostKeyChecking=no -r temp_repo $EC2_HOST:~/temp_repo
        
                        echo ' Running Dependency-Check on EC2...'
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
        
                        echo ' Copying report back to Jenkins workspace...'
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
                            docker system prune -f
                        '''
                    }
                }
            }
        }
        
        // running
        stage('Deploy App to AWS EC2') {
            steps {
                echo ' Deploying Juice Shop to EC2...'
                sshagent(credentials: [env.EC2_KEY_ID]) {
                     sh """#!/bin/bash
                        ssh -o StrictHostKeyChecking=no $WEB_HOST << 'ENDSSH'
                            echo "🔍 Checking for any process on port $WEB_APP_PORT..."
                            PID=\$(sudo lsof -t -i:$WEB_APP_PORT)
                            if [ ! -z "\$PID" ]; then
                                echo " Killing process \$PID using port $WEB_APP_PORT"
                                sudo kill -9 \$PID || true
                            else
                                echo "✅ No process found on port $WEB_APP_PORT"
                            fi
        
                            echo " Removing old Docker container..."
                            docker rm -f juice-shop || true
        
                            echo " Pulling Docker image: $IMAGE_NAME:$TAG"
                            docker pull $IMAGE_NAME:$TAG
        
                            echo " Running new Docker container..."
                            docker run -d -p $WEB_APP_PORT:$WEB_APP_PORT --name juice-shop $IMAGE_NAME:$TAG
        
                            echo "✅ Deployment complete"
                        ENDSSH
                        sleep 20
                    """
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
                            nikto -h $TARGET_URL -output ~/zap-work/nikto_report.xml -Format xml \\
                            -Display V \\
                            -Plugins ALL \\
                            -Tuning 1234567890 \\
                            -Cgidirs all \\
                            -no404 
                        '
                        echo " Copying Nikto report to Jenkins workspace..."
                        scp -o StrictHostKeyChecking=no $ZAP_INSTANCE_HOST:~/zap-work/nikto_report.xml .
                    """
                    archiveArtifacts artifacts: 'nikto_report.xml', onlyIfSuccessful: false
                }
            }
        }
        
         stage('Upload TruffleHog Report to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'DEFECTDOJO_API_TOKEN', variable: 'DD_API_KEY')]) {
                    sh '''
                        if [ -f trufflehog_report.json ]; then
                            curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                              -H "Authorization: Token $DD_API_KEY" \
                              -F "file=@trufflehog_report.json" \
                              -F "scan_type=Trufflehog Scan" \
                              -F "engagement=$ENGAGEMENT_ID" \
                              -F "active=true" -F "verified=true" -F "close_old_findings=true"
                        fi
                    '''
                }
            }
        }
        
        
        stage('Upload Dependency-Check Report to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'DEFECTDOJO_API_TOKEN', variable: 'DD_API_KEY')]) {
                    sh '''
                        if [ -f dependency-check-report.json ]; then
                            curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                              -H "Authorization: Token $DD_API_KEY" \
                              -F "file=@dependency-check-report.json" \
                              -F "scan_type=Dependency Check Scan" \
                              -F "engagement=$ENGAGEMENT_ID" \
                              -F "active=true" -F "verified=true" -F "close_old_findings=true"
                        fi
                    '''
                }
            }
        } 
              
        stage('Upload ZAP Report to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'DEFECTDOJO_API_TOKEN', variable: 'DD_API_KEY')]) {
                    sh '''
                        if [ -f $ZAP_REPORT_XML ]; then
                            curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                              -H "Authorization: Token $DD_API_KEY" \
                              -F "file=@$ZAP_REPORT_XML" \
                              -F "scan_type=ZAP Scan" \
                              -F "engagement=$ENGAGEMENT_ID" \
                              -F "active=true" -F "verified=true" -F "close_old_findings=true"
                        fi
                    '''
                }
            }
        }
        
        stage('Upload Nikto Report to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'DEFECTDOJO_API_TOKEN', variable: 'DD_API_KEY')]) {
                    sh '''
                        if [ -f nikto_report.xml ]; then
                            curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                              -H "Authorization: Token $DD_API_KEY" \
                              -F "file=@nikto_report.xml" \
                              -F "scan_type=Nikto Scan" \
                              -F "engagement=$ENGAGEMENT_ID" \
                              -F "active=true" \
                              -F "verified=true" \
                              -F "close_old_findings=true"
                        fi
                    '''
                }
            }
        }

        
    }

    post {
        always {
            script {
                echo 'Cleaning up temporary files...'
                sh '''
                    rm -rf temp_repo dependency-check-report trufflehog_report.json trufflehog_report.txt nikto_report.html juice-shop \
                           $ZAP_REPORT_HTML $ZAP_REPORT_XML $ZAP_REPORT_JSON || true
                '''
            }
        }
    }
}
