// =============================================================================
// Pipeline CI/CD do Distrimed.
//
// Fluxo:
//   Checkout (+ submodules)
//     → Backend: install + lint + test
//     → Frontend: install + lint + test
//     → Build images (paralelo)
//     → Trivy scan (paralelo, HIGH/CRITICAL + --ignore-unfixed)
//     → Push GHCR (apenas main)
//     → Deploy local prod (apenas main)
//
// Pré-requisitos no Jenkins:
//   - Credencial "ghcr-pat" (Username/Password) — username = login GitHub,
//     password = PAT com escopo write:packages.
//   - Credencial "jwt-secret-prod" (Secret Text) — JWT_SECRET do deploy.
//   - Docker CLI disponível no controller (DooD via socket do host).
// =============================================================================

pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        GHCR_OWNER     = 'eidrianxd'
        IMAGE_BACKEND  = "ghcr.io/${GHCR_OWNER}/distrimed-backend"
        IMAGE_FRONTEND = "ghcr.io/${GHCR_OWNER}/distrimed-frontend"
    }

    triggers {
        // Polling a cada minuto (com jitter "H"). Suficiente para demo local.
        // Em produção real, prefira webhook do GitHub.
        pollSCM('H/1 * * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --init --recursive'
                script {
                    env.SHA_SHORT = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short=7 HEAD'
                    ).trim()
                }
                echo "Build do commit ${env.SHA_SHORT} (branch ${env.BRANCH_NAME ?: 'unknown'})"
            }
        }

        stage('Backend: install + lint + test') {
            agent {
                docker {
                    image 'node:20-slim'
                    reuseNode true
                }
            }
            steps {
                dir('backend') {
                    sh 'npm ci --no-audit --no-fund'
                    sh 'npm run lint'
                    sh 'npm test'
                }
            }
        }

        stage('Frontend: install + lint + test') {
            agent {
                docker {
                    image 'node:22-slim'
                    reuseNode true
                }
            }
            steps {
                dir('frontend') {
                    sh 'npm ci --no-audit --no-fund'
                    sh 'npm run lint'
                    sh 'npm test'
                }
            }
        }

        stage('Build images') {
            parallel {
                stage('backend') {
                    steps {
                        sh '''
                            docker build \
                                -t $IMAGE_BACKEND:$SHA_SHORT \
                                -t $IMAGE_BACKEND:latest \
                                backend/
                        '''
                    }
                }
                stage('frontend') {
                    steps {
                        sh '''
                            docker build \
                                --build-arg VITE_API_BASE_URL=/api \
                                -t $IMAGE_FRONTEND:$SHA_SHORT \
                                -t $IMAGE_FRONTEND:latest \
                                frontend/
                        '''
                    }
                }
            }
        }

        stage('Trivy scan') {
            // --skip-dirs ignora o `npm` bundled na imagem base do Node
            // (usr/local/lib/node_modules/npm). Esse npm é uma ferramenta
            // de instalação que vem com a imagem oficial node:*-slim e NÃO
            // é executado em runtime (o backend roda `node dist/src/main.js`).
            // CVEs ali são ruído — não há vetor de exploração no nosso runtime.
            //
            // Cada branch paralela usa seu PRÓPRIO volume de cache. O cache do
            // Trivy usa BoltDB com lock exclusivo durante a init; compartilhar
            // entre scans paralelos gera "cache may be in use by another process".
            // Custo: ~250 MB extras de disco no host (cada cache baixa o DB).
            parallel {
                stage('backend') {
                    steps {
                        sh '''
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v trivy-cache-backend:/root/.cache/ \
                                aquasec/trivy:latest image \
                                --severity HIGH,CRITICAL \
                                --ignore-unfixed \
                                --skip-dirs usr/local/lib/node_modules/npm \
                                --exit-code 1 \
                                --no-progress \
                                --timeout 5m \
                                $IMAGE_BACKEND:$SHA_SHORT
                        '''
                    }
                }
                stage('frontend') {
                    steps {
                        sh '''
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v trivy-cache-frontend:/root/.cache/ \
                                aquasec/trivy:latest image \
                                --severity HIGH,CRITICAL \
                                --ignore-unfixed \
                                --skip-dirs usr/local/lib/node_modules/npm \
                                --exit-code 1 \
                                --no-progress \
                                --timeout 5m \
                                $IMAGE_FRONTEND:$SHA_SHORT
                        '''
                    }
                }
            }
        }

        stage('Push GHCR') {
            when {
                anyOf {
                    branch 'main'
                    expression { env.BRANCH_NAME == null }  // job não-multibranch
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ghcr-pat',
                    usernameVariable: 'GHCR_USER',
                    passwordVariable: 'GHCR_TOKEN'
                )]) {
                    sh '''
                        echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
                        docker push $IMAGE_BACKEND:$SHA_SHORT
                        docker push $IMAGE_BACKEND:latest
                        docker push $IMAGE_FRONTEND:$SHA_SHORT
                        docker push $IMAGE_FRONTEND:latest
                    '''
                    // Logout fica em post { always } para garantir que rode
                    // mesmo se o deploy falhar — assim as creds do GHCR não
                    // ficam persistidas no docker config do controller.
                }
            }
        }

        stage('Deploy local prod') {
            when {
                anyOf {
                    branch 'main'
                    expression { env.BRANCH_NAME == null }
                }
            }
            environment {
                COMPOSE_PROJECT_NAME = 'distrimed-prod'
                BACKEND_TAG          = "${env.SHA_SHORT}"
                FRONTEND_TAG         = "${env.SHA_SHORT}"
                // SEED_ON_BOOT=true neste deploy de portfolio para garantir
                // que a aplicação prod fique imediatamente demonstrável (login
                // john/123456 funciona). Em produção real, este seed seria
                // executado uma única vez como job separado.
                SEED_ON_BOOT         = 'true'
                FRONTEND_PORT        = '8090'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'jwt-secret-prod', variable: 'JWT_SECRET'),
                    string(credentialsId: 'db-password-prod', variable: 'DB_PASSWORD'),
                    usernamePassword(
                        credentialsId: 'ghcr-pat',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                        # Re-autentica caso o GHCR ainda esteja privado (caso
                        # comum logo após o primeiro push de um repo novo).
                        echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
                        docker compose -p $COMPOSE_PROJECT_NAME pull
                        docker compose -p $COMPOSE_PROJECT_NAME up -d --no-build
                    '''
                }
            }
        }
    }

    post {
        always {
            // Limpa credenciais do registry e imagens dangling. `|| true`
            // protege o resultado do build caso algum prune falhe.
            sh 'docker logout ghcr.io || true'
            sh 'docker image prune -f --filter "until=24h" || true'
        }
        success {
            echo "Build #${env.BUILD_NUMBER} (commit ${env.SHA_SHORT}) — sucesso."
        }
        failure {
            echo "Build #${env.BUILD_NUMBER} (commit ${env.SHA_SHORT}) — falhou."
        }
    }
}
