pipeline {
    agent any

    environment {
        DEPENDENCY_CHECK = '/home/rocky1/tools/dependency-check/bin/dependency-check.sh'
        //SONAR_SCANNER = tool name: 'sonar-scanner'
        ZAP_REPORT_HTML = 'zap_report.html'
        ZAP_REPORT_XML  = 'zap_report.xml'
        ZAP_REPORT_JSON = 'zap_report.json'
        TARGET_URL      = 'http://localhost:3000' // Replace with actual target
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
        // Trufflehog is installed locally
        stage('Secret Scan (TruffleHog)') {
            steps {
                echo 'Running TruffleHog on latest commit only...'
                sh '''
                  cd temp_repo
                  trufflehog --regex --entropy=True --max_depth=10 . > ../trufflehog_report.json || true
                  cd ..
                '''
                archiveArtifacts artifacts: 'trufflehog_report.json', onlyIfSuccessful: false
            }
        }
/*      THis will work only when you have installed dependecy check locally
        stage('Dependency Check (OWASP)') {
            steps {
                echo 'Running OWASP Dependency-Check...'
                sh '''
                    mkdir -p dependency-check-report
                    cd temp_repo
                    $DEPENDENCY_CHECK --project "Universal-SCA-Scan" --scan . --format ALL --out ../dependency-check-report || true
                    cd ..
                '''
                archiveArtifacts artifacts: 'dependency-check-report/*', onlyIfSuccessful: false
            }
        }
        */
        
        /* This will work only when NVD database start accepting api keys
        stage('Dependency Check (OWASP)') {
            steps {
                echo 'Running OWASP Dependency-Check using Docker...'
                withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_KEY')]) {
                    sh '''
                        mkdir -p dependency-check-report
                        echo "NVD API Key is: ${NVD_KEY:0:4}********"
                        curl -I https://services.nvd.nist.gov/rest/json/cves/2.0
                        docker run --rm \
                          -v $WORKSPACE/temp_repo:/src \
                          -v $WORKSPACE/dependency-check-report:/report \
                          owasp/dependency-check \
                          --nvdApiKey $NVD_KEY \
                          --project Universal-SCA-Scan \
                          --scan /src \
                          --format ALL \
                          --out /report || true
                    '''
                }
                archiveArtifacts artifacts: 'dependency-check-report/*', onlyIfSuccessful: false
            }
        }
        */ 
        // This will work using docker image of dependency check without using nvd aoi key
        stage('Dependency Check (OWASP via Docker)') {
            steps {
                echo 'Running OWASP Dependency-Check using Docker...'
                sh '''
                    mkdir -p dependency-check-report
                    docker run --rm \
                      -v $WORKSPACE/temp_repo:/src \
                      -v $WORKSPACE/dependency-check-report:/report \
                      owasp/dependency-check \
                      --project "Universal-SCA-Scan" \
                      --scan /src \
                      --exclude **/*.zip \
                      --exclude "../../*" \
                      --enableExperimental \
                      --format ALL \
                      --out /report || true
                '''
                archiveArtifacts artifacts: 'dependency-check-report/*', onlyIfSuccessful: false
            }
        }

        

        

        /* 
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
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }
        */
/*
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
                echo 'Building Docker Image...'
                dir('temp_repo') {
                    sh 'docker build -t juice-shop .'
                }
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                echo 'Deploying Docker container to EC2...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@13.51.168.90 << EOF
                            docker stop juice-shop || true
                            docker rm juice-shop || true
                            docker pull kumar0ndocker/juice-shop
                            docker run -d --name juice-shop -p 3000:3000 kumar0ndocker/juice-shop
                        EOF
                    '''
                }
            }
        }
*/
        /*
        stage('Deploy to Server') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    sshagent(credentials: ['app-server']) {
                        sh '''
                            scp -o StrictHostKeyChecking=no temp_repo/webgoat-server/target/webgoat-server-v8.2.0-SNAPSHOT.jar ubuntu@3.109.152.116:/WebGoat
                            ssh -o StrictHostKeyChecking=no ubuntu@3.109.152.116 "nohup java -jar /WebGoat/webgoat-server-v8.2.0-SNAPSHOT.jar > /dev/null 2>&1 &"
                        '''
                    }
                }
            }
        }
        */

        stage('Run ZAP DAST Scan (Baseline)') {
            steps {
                echo 'Running ZAP Baseline DAST Scan...'
                sh '''
                    docker run --rm \
                      -v $WORKSPACE:/zap/wrk/:rw \
                      zaproxy/zap-stable \
                      zap-baseline.py -t $TARGET_URL \
                      -r $ZAP_REPORT_HTML -x $ZAP_REPORT_XML -J $ZAP_REPORT_JSON || true
                '''
                archiveArtifacts artifacts: "${ZAP_REPORT_HTML}, ${ZAP_REPORT_XML}, ${ZAP_REPORT_JSON}", onlyIfSuccessful: false
            }
        }
    }

    post {
        always {
            script {
                echo 'Cleaning up temporary files...'
                sh '''
                    rm -rf temp_repo dependency-check-report trufflehog_report.txt \
                           $ZAP_REPORT_HTML $ZAP_REPORT_XML $ZAP_REPORT_JSON || true
                '''
            }
        }
    }
}
