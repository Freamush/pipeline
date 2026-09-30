pipeline {
    agent {
        docker { 
            image 'misc.rof.local:5000/php-ci:latest'
            registryUrl 'http://misc.rof.local:5000'
            args '''
            --network host
            -v $HOME/.composer:/tmp/composer-cache
            -v /var/jenkins_home/tools:/var/jenkins_home/tools
            '''
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        gitLabConnection('honeypot-project')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    updateGitlabCommitStatus name: 'checkout', state: 'running'
                    try {
                        checkout scm
                        updateGitlabCommitStatus name: 'checkout', state: 'success'
                    } catch (e) {
                        updateGitlabCommitStatus name: 'checkout', state: 'failed'
                        throw e
                    }
                }
            }
        }

        stage('Install') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        updateGitlabCommitStatus name: 'install', state: 'running'
                        try {
                            sh '''
                                rm -rf build/
                                mkdir -p build
                                composer config cache-dir /tmp/composer-cache
                                composer install \
                                    --no-interaction \
                                    --prefer-dist \
                                    --ignore-platform-reqs
                            '''
                            updateGitlabCommitStatus name: 'install', state: 'success'
                        } catch (e) {
                            updateGitlabCommitStatus name: 'install', state: 'failed'
                            throw e
                        }
                    }
                }
            }
        }

        stage('PHPStan') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        updateGitlabCommitStatus name: 'phpstan', state: 'running'
        
                        int status = sh(
                            script: '''
                            php vendor/bin/phpstan analyse \
                                --error-format=json \
                                --no-progress \
                                2>/dev/null \
                                > build/phpstan-report.json
                            ''',
                            returnStatus: true
                        )
        
                        if (status != 0) {
                            unstable('PHPStan issues detected')
                            updateGitlabCommitStatus name: 'phpstan', state: 'failed'
                        } else {
                            updateGitlabCommitStatus name: 'phpstan', state: 'success'
                        }
                    }
                }
            }
        }

                stage('Psalm') {
                    steps {
                        timeout(time: 5, unit: 'MINUTES') {
                            script {
                                updateGitlabCommitStatus name: 'psalm', state: 'running'

                                sh 'mkdir -p build'

                                int status = sh(
                                    script: '''
                                        php vendor/bin/psalm \
                                            --taint-analysis \
                                            --no-progress \
                                            --report=build/psalm-report.sonarqube.json \
                                            --report-show-info=false
                                    ''',
                                    returnStatus: true
                                )

                                if (status != 0) {
                                    unstable('Psalm issues detected')
                                    updateGitlabCommitStatus name: 'psalm', state: 'failed'
                                } else {
                                    updateGitlabCommitStatus name: 'psalm', state: 'success'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    script {
                        updateGitlabCommitStatus name: 'owasp', state: 'running'
                        try {
                            dependencyCheck(
                                additionalArguments: '--scan . ' +
                                    '--exclude \'vendor/**\' ' +
                                    '--out build/ ' +
                                    '--disableYarnAudit ' +
                                    '--disableNodeAudit',
                                nvdCredentialsId: 'NVD_KEY', 
                                odcInstallation: 'OWASP-DC'
                            )
                            dependencyCheckPublisher(
                                pattern: '**/dependency-check-report.xml',
                                failedTotalCritical: 1,
                                unstableTotalHigh: 5
                            )
                            updateGitlabCommitStatus name: 'owasp', state: 'success'
                        } catch (e) {
                            updateGitlabCommitStatus name: 'owasp', state: 'failed'
                            throw e
                        }
                    }
                }
            }
        }

        stage('SonarQube') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        updateGitlabCommitStatus name: 'sonar', state: 'running'
                        try {
                            withSonarQubeEnv('SonarQube') {
                                sh '''
                                    sonar-scanner \
                                    -Dsonar.host.url=http://sonarqube:9000 \
                                    -Dsonar.projectKey=honeypot-php \
                                    -Dsonar.sources=public,includes \
                                    -Dsonar.exclusions=vendor/**,build/**,logs/** \
                                    -Dsonar.php.phpstan.reportPaths=build/phpstan-report.json \
                                    -Dsonar.php.psalm.reportPaths=build/psalm-report.sonarqube.json \
                                    -X
                                '''
                            }
                            timeout(time: 5, unit: 'MINUTES') {
                                def qg = waitForQualityGate(abortPipeline: false)
                                env.SONAR_GATE = qg.status
                                echo "Quality Gate: ${env.SONAR_GATE}"
                                if (qg.status != 'OK') {
                                    error("Quality Gate failed: ${qg.status}")
                                }
                            }

                            updateGitlabCommitStatus name: 'sonar', state: 'success'
                        } catch (e) {
                            updateGitlabCommitStatus name: 'sonar', state: 'failed'
                            throw e
                        }
                    }
                }
            }
        }      
    }

    post {
        success {
            updateGitlabCommitStatus name: 'pipeline', state: 'success'
        }
        failure {
            updateGitlabCommitStatus name: 'pipeline', state: 'failed'
        }
        always { 
            archiveArtifacts artifacts: "build/*", fingerprint: true
        }
    }
}
