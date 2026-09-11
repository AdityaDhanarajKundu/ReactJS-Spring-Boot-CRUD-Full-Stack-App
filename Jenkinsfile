// ============================================================================
//  Employee App - CI/CD  (React + Spring Boot -> DockerHub -> GitOps -> Argo CD)
//
//  Every stage runs inside a throwaway Docker container ON THE WORKER NODE.
//  The controller has 0 executors; the worker has nothing installed but Docker.
//
//  TUNED FOR A 20 GB WORKER DISK:
//    - Trivy DB cached on the host (else ~700 MB re-downloaded every build)
//    - tagged images deleted locally after push (only :latest kept as warm cache)
//    - disk guard fails fast instead of dying mid-build with ENOSPC
//
//  BEFORE FIRST RUN edit:
//    1. the four constants under `environment`
//    2. the --group-add value in `args`  ->  getent group docker | cut -d: -f3
// ============================================================================

pipeline {

    agent {
        docker {
            label 'worker'
            image 'adityadhanarajkundu/jenkins-fullstack-agent:v1'
            args  '''
                -v /var/run/docker.sock:/var/run/docker.sock
                --group-add 986
                -v /opt/jenkins-cache/.m2:/cache/.m2
                -v /opt/jenkins-cache/.npm:/cache/.npm
                -v /opt/jenkins-cache/trivy:/cache/trivy
            '''
            reuseNode false
            alwaysPull false
        }
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()
        skipDefaultCheckout(false)
    }

    // Uncomment to poll instead of using a GitHub webhook (keeps Jenkins private):
    triggers { pollSCM('H/3 * * * *') }

    environment {
        // ------------------- EDIT THESE FOUR -------------------
        DOCKERHUB_USER = 'adityadhanarajkundu'
        GITOPS_REPO    = 'github.com/AdityaDhanarajKundu/employee-gitops.git'
        ARGOCD_SERVER  = '172.31.5.234:30080'      // <KIND_PRIVATE_IP>:30080
        SONAR_ENV_NAME = 'SonarQube'           // must match Manage Jenkins > System
        // -------------------------------------------------------

        BACKEND_IMAGE    = "${DOCKERHUB_USER}/employee-backend"
        FRONTEND_IMAGE   = "${DOCKERHUB_USER}/employee-frontend"
        MAVEN_OPTS       = '-Dmaven.repo.local=/cache/.m2 -Xmx768m'
        NPM_CONFIG_CACHE = '/cache/.npm'
        TRIVY_CACHE_DIR  = '/cache/trivy'
        MIN_FREE_GB      = '4'
    }

    stages {

        stage('Init & Disk Guard') {
            steps {
                script {
                    env.GIT_SHA   = sh(returnStdout: true, script: 'git rev-parse --short=8 HEAD').trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} - ${env.GIT_SHA}"
                }
                sh '''
                    echo "==================== build agent ===================="
                    whoami && id
                    java -version 2>&1 | head -1
                    mvn -v | head -1
                    node -v && npm -v
                    docker version --format 'docker client {{.Client.Version}} / server {{.Server.Version}}'
                    echo "image tag: ${IMAGE_TAG}"

                    echo "==================== disk guard ===================="
                    df -h /
                    FREE_GB=$(df -BG --output=avail / | tail -1 | tr -dc '0-9')
                    echo "free on / : ${FREE_GB} GB   (minimum ${MIN_FREE_GB} GB)"
                    if [ "$FREE_GB" -lt "$MIN_FREE_GB" ]; then
                        echo ""
                        echo "ABORTING: worker is low on disk."
                        echo "  docker system prune -af --volumes"
                        echo "  rm -rf /opt/jenkins-cache/.npm/*"
                        exit 1
                    fi
                '''
            }
        }

        stage('Backend - Build & Test') {
            steps {
                dir('springboot-backend') {
                    sh 'mvn -B -ntp clean verify'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'springboot-backend/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Backend - SonarQube') {
            steps {
                dir('springboot-backend') {
                    withSonarQubeEnv("${SONAR_ENV_NAME}") {
                        sh '''
                            mvn -B -ntp sonar:sonar \
                              -Dsonar.projectKey=employee-backend \
                              -Dsonar.projectName="Employee Backend" \
                              -Dsonar.projectVersion=${IMAGE_TAG} \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                        '''
                    }
                }
            }
        }

        stage('Backend - Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Frontend - Install & Build') {
            steps {
                dir('react-frontend') {
                    sh '''
                        if [ -f package-lock.json ]; then
                            npm ci --no-audit --no-fund
                        else
                            npm install --no-audit --no-fund
                        fi

                        CI=true npm test -- --watchAll=false --passWithNoTests \
                            --coverage --coverageReporters=lcov --coverageReporters=text-summary || true

                        export NODE_OPTIONS=--openssl-legacy-provider
                        CI=false npm run build
                    '''
                }
    }
}

        stage('Frontend - SonarQube') {
            steps {
                dir('react-frontend') {
                    withSonarQubeEnv("${SONAR_ENV_NAME}") {
                        sh 'sonar-scanner -Dsonar.projectVersion=${IMAGE_TAG}'
                    }
                }
            }
        }

        stage('Frontend - Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -f springboot-backend/Dockerfile.backend.ci \
                                 -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                                 -t ${BACKEND_IMAGE}:latest \
                                 springboot-backend

                    docker build -f react-frontend/Dockerfile.frontend.ci \
                                 -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                                 -t ${FRONTEND_IMAGE}:latest \
                                 react-frontend

                    docker images | grep employee-
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    # --cache-dir is host-mounted: the ~700 MB vuln DB downloads once,
                    # not on every build. Without it this stage dominates build time.
                    trivy image --cache-dir ${TRIVY_CACHE_DIR} \
                          --scanners vuln --severity HIGH,CRITICAL \
                          --exit-code 0 --no-progress --format table \
                          ${BACKEND_IMAGE}:${IMAGE_TAG} | tee trivy-backend.txt

                    trivy image --cache-dir ${TRIVY_CACHE_DIR} \
                          --scanners vuln --severity HIGH,CRITICAL \
                          --exit-code 0 --no-progress --format table \
                          ${FRONTEND_IMAGE}:${IMAGE_TAG} | tee trivy-frontend.txt
                '''
            }
            post {
                always { archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true }
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds',
                                                  usernameVariable: 'GH_USER',
                                                  passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        rm -rf gitops
                        git clone --depth 1 "https://${GH_USER}:${GH_TOKEN}@${GITOPS_REPO}" gitops

                        cd gitops/apps/employee
                        kustomize edit set image ${BACKEND_IMAGE}=${BACKEND_IMAGE}:${IMAGE_TAG}
                        kustomize edit set image ${FRONTEND_IMAGE}=${FRONTEND_IMAGE}:${IMAGE_TAG}

                        cd "${WORKSPACE}/gitops"
                        git config user.email "jenkins@ci.local"
                        git config user.name  "jenkins"

                        if git diff --quiet; then
                            echo "no manifest change - images already at ${IMAGE_TAG}"
                        else
                            git add -A
                            git commit -m "ci: employee images -> ${IMAGE_TAG} (build ${BUILD_NUMBER})"
                            git push origin HEAD:main
                        fi
                    '''
                }
            }
        }

        stage('Argo CD Sync & Wait') {
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_AUTH_TOKEN')]) {
                    sh '''
                        argocd app sync employee \
                            --server ${ARGOCD_SERVER} --plaintext --grpc-web --timeout 300

                        argocd app wait employee \
                            --server ${ARGOCD_SERVER} --plaintext --grpc-web \
                            --health --sync --timeout 420

                        argocd app get employee --server ${ARGOCD_SERVER} --plaintext --grpc-web
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "OK  ${IMAGE_TAG} deployed -> http://<KIND_PUBLIC>:30081"
        }
        failure {
            echo "FAILED  build ${BUILD_NUMBER}"
        }
        always {
            // 20 GB disk hygiene: the tagged images are safely on DockerHub now,
            // so drop the local copies. :latest stays as a layer-cache warm start.
            sh '''
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG}  || true
                docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                docker image prune -f || true
                echo "--- disk after build ---"
                df -h /
            '''
        }
    }
}
