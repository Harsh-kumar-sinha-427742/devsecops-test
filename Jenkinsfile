pipeline {
    agent any

    environment {
        DEPENDENCY_CHECK = '/home/rocky1/tools/dependency-check/bin/dependency-check.sh'
        SONAR_SCANNER = tool name: 'sonar-scanner'
        ZAP_REPORT_HTML = 'zap_report.html'
        ZAP_REPORT_XML  = 'zap_report.xml'
        ZAP_REPORT_JSON = 'zap_report.json'
        TARGET_URL      = 'http://localhost:3000' // Replace with actual target
    }

    stages {
        stage('Clone Repository') {
            steps {
                echo 'Cloning the GitHub Repository...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        rm -rf temp_repo
                        git clone --depth=1 https://github.com/Akashsonawane571/devsecops-test.git temp_repo
                    '''
                }
            }
        }

        stage('Secret Scan (TruffleHog)') {
            steps {
                echo 'Running TruffleHog on latest commit only...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                      cd temp_repo
                      trufflehog --regex --entropy=True --max_depth=10 . > ../trufflehog_report.json
                      cd ..
                    '''
                }
                archiveArtifacts artifacts: 'trufflehog_report.json', onlyIfSuccessful: false
            }
        }

        stage('Dependency Check (OWASP)') {
            steps {
                echo 'Running OWASP Dependency-Check...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        mkdir -p dependency-check-report
                        cd temp_repo
                        $DEPENDENCY_CHECK --project "Universal-SCA-Scan" --scan . --format ALL --out ../dependency-check-report
                        cd ..
                    '''
                }
                archiveArtifacts artifacts: 'dependency-check-report/*', onlyIfSuccessful: false
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo 'Starting SonarQube SAST Scan...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
        }

        stage('Build Project') {
            steps {
                echo 'Building the Java project with Maven...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    dir('temp_repo') {
                        sh 'mvn clean install -DskipTests'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    dir('temp_repo') {
                        sh 'docker build -t juice-shop .'
                    }
                }
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                echo 'Deploying Docker container to EC2...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
        }

        stage('Run ZAP DAST Scan (Baseline)') {
            steps {
                echo 'Running ZAP Baseline DAST Scan...'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        docker run --rm \
                          -v $WORKSPACE:/zap/wrk/:rw \
                          zaproxy/zap-stable \
                          zap-baseline.py -t $TARGET_URL \
                          -r $ZAP_REPORT_HTML -x $ZAP_REPORT_XML -J $ZAP_REPORT_JSON
                    '''
                }
                archiveArtifacts artifacts: "${ZAP_REPORT_HTML}, ${ZAP_REPORT_XML}, ${ZAP_REPORT_JSON}", onlyIfSuccessful: false
            }
        }
    }

    post {
        always {
            echo 'Cleaning up temporary files...'
            sh '''
                rm -rf temp_repo dependency-check-report trufflehog_report.txt \
                       $ZAP_REPORT_HTML $ZAP_REPORT_XML $ZAP_REPORT_JSON || true
            '''
        }
    }
}
