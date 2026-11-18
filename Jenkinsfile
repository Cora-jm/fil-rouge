pipeline {
    agent any

    environment {
        IMAGE_REPO = 'coraliejm/legeni-webapp'
        TEST_IMAGE = "legeni-test:${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Préparation') {
            steps {
                script {
                    env.BASE_VERSION = readFile('VERSION').trim()
                    env.IMAGE_TAG = "${env.BASE_VERSION}-${env.BUILD_NUMBER}"
                }
                echo "Image visée : ${IMAGE_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Image de test') {
            steps {
                sh 'docker build --target test -t $TEST_IMAGE .'
            }
        }

        stage('Contrôles qualité') {
            parallel {
                stage('Tests et couverture') {
                    steps {
                        sh '''
                            set +e
                            mkdir -p reports
                            docker rm -f tests-$BUILD_NUMBER >/dev/null 2>&1
                            docker run --name tests-$BUILD_NUMBER $TEST_IMAGE \
                                pytest -q --junitxml=/app/junit.xml --cov=app \
                                --cov-report=xml:/app/coverage.xml --cov-fail-under=80
                            rc=$?
                            docker cp tests-$BUILD_NUMBER:/app/junit.xml reports/junit.xml
                            docker cp tests-$BUILD_NUMBER:/app/coverage.xml reports/coverage.xml
                            docker rm -f tests-$BUILD_NUMBER
                            exit $rc
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                            archiveArtifacts artifacts: 'reports/*.xml', allowEmptyArchive: true
                        }
                    }
                }
                stage('Style (flake8)') {
                    steps {
                        sh 'docker run --rm $TEST_IMAGE flake8 .'
                    }
                }
                stage('Dockerfile (hadolint)') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint < Dockerfile'
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        dir('ansible') {
                            sh '''
                                ansible-playbook --syntax-check provision.yml
                                ansible-playbook --syntax-check deploy.yml -e target_env=staging -e image_tag=0.0.0
                            '''
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$IMAGE_TAG -t $IMAGE_REPO:$IMAGE_TAG .'
            }
        }

        stage('Scan Trivy') {
            steps {
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy-cache:/root/.cache/ \
                        aquasec/trivy:latest image \
                        --exit-code 1 --severity CRITICAL --ignore-unfixed \
                        $IMAGE_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Push Docker Hub') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE_REPO:$IMAGE_TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 string(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
                    dir('ansible') {
                        sh '''
                            VP=$(mktemp)
                            trap 'rm -f "$VP"' EXIT
                            printf '%s' "$VAULT_PASS" > "$VP"
                            ansible-playbook deploy.yml -e target_env=staging -e image_tag=$IMAGE_TAG --vault-password-file "$VP"
                        '''
                    }
                }
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Déployer ${IMAGE_TAG} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 string(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
                    dir('ansible') {
                        sh '''
                            VP=$(mktemp)
                            trap 'rm -f "$VP"' EXIT
                            printf '%s' "$VAULT_PASS" > "$VP"
                            ansible-playbook deploy.yml -e target_env=prod -e image_tag=$IMAGE_TAG --vault-password-file "$VP"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker rmi -f $TEST_IMAGE >/dev/null 2>&1 || true'
        }
        success {
            echo "Pipeline terminé : ${IMAGE_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "Échec du pipeline : consulte l'étape en rouge"
        }
    }
}
