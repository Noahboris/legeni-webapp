// Lot 4 : intégration continue.
//   qualité (4 contrôles en parallèle) -> build -> test d'acceptance -> scan Trivy -> push (main uniquement)
// Aucun secret dans ce fichier : uniquement des credentials Jenkins
//   registry-credentials (Docker Hub), ansible-vault-pass (coffre Ansible).
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
        disableConcurrentBuilds()
    }

    environment {
        IMAGE          = 'noahboris/legeni-webapp'
        TEST_CONTAINER = "webapp-${env.BUILD_TAG}"
        CI_NETWORK     = "net-${env.BUILD_TAG}"
    }

    stages {
        stage('Préparation') {
            steps {
                script {
                    env.APP_VERSION = readFile('VERSION').trim()
                    // Le tag de l'image est toujours <VERSION>-<numéro de build> : jamais "latest"
                    env.TAG = "${env.APP_VERSION}-${env.BUILD_NUMBER}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.TAG}"
                }
                sh '''
                    grep -Eq '^[0-9]+\\.[0-9]+\\.[0-9]+$' VERSION
                    mkdir -p reports
                    python3 -m venv .venv
                    .venv/bin/pip install --quiet --no-cache-dir -r requirements.txt -r requirements-dev.txt
                '''
            }
        }

        stage('Qualité') {
            failFast false
            parallel {
                stage('Tests + couverture') {
                    steps {
                        sh '''
                            .venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml --cov-report=term \
                                --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('flake8') {
                    steps {
                        sh '.venv/bin/flake8 . --output-file=reports/flake8.txt --tee'
                    }
                    post {
                        always {
                            recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')])
                        }
                    }
                }
                stage('hadolint') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint hadolint --format json - < Dockerfile > reports/hadolint.json'
                    }
                    post {
                        always {
                            recordIssues(tools: [hadoLint(pattern: 'reports/hadolint.json')])
                        }
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        withCredentials([file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                            sh '''
                                cd ansible
                                export ANSIBLE_VAULT_PASSWORD_FILE="$VAULT_FILE"
                                ansible-playbook provision.yml --syntax-check
                                ansible-playbook deploy.yml --syntax-check -e target_env=staging -e app_version=0.0.0-0
                            '''
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --pull --build-arg APP_VERSION=$TAG -t $IMAGE:$TAG .'
            }
        }

        stage("Test d'acceptance") {
            steps {
                sh '''
                    docker network create $CI_NETWORK
                    docker run -d --name $TEST_CONTAINER --network $CI_NETWORK $IMAGE:$TAG
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf --retry 10 --retry-connrefused --retry-delay 1 http://$TEST_CONTAINER:5000/ready
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf http://$TEST_CONTAINER:5000/api/info | grep -q "\\"version\\":\\"$TAG\\""
                '''
            }
        }

        stage('Scan Trivy') {
            steps {
                sh '''
                    TRIVY="docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:latest"
                    # Rapport complet (HIGH + CRITICAL), qui ne bloque pas
                    $TRIVY image --no-progress --severity HIGH,CRITICAL --ignore-unfixed $IMAGE:$TAG > reports/trivy-report.txt || true
                    cat reports/trivy-report.txt
                    # Porte de qualité : aucune vulnérabilité CRITICAL corrigeable
                    $TRIVY image --no-progress --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$TAG
                '''
            }
        }

        stage('Push Docker Hub') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER',
                                                  passwordVariable: 'REG_TOKEN')]) {
                    sh '''
                        echo "$REG_TOKEN" | docker login -u "$REG_USER" --password-stdin
                        docker push $IMAGE:$TAG
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f $TEST_CONTAINER || true; docker network rm $CI_NETWORK || true'
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
        success {
            echo "Image ${env.IMAGE}:${env.TAG} construite et validée"
        }
        failure {
            echo "Échec du pipeline : consultez l'étape en rouge"
        }
    }
}
