pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        PNPM_HOME      = "${HOME}/.local/share/pnpm"
        PATH           = "${PNPM_HOME}:${PATH}"
        NODE_ENV       = 'test'
        // Secretos gestionados desde Jenkins Credentials (tipo Secret text)
        JWT_SECRET     = credentials('canipet-jwt-secret')
        DB_PASSWORD    = credentials('canipet-db-password')
    }

    stages {

        // ── 0. Obtener código ───────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Rama: ${env.BRANCH_NAME} · Commit: ${env.GIT_COMMIT?.take(8)}"
            }
        }

        // ── 1. Backend ──────────────────────────────────────────────────────
        stage('Backend') {
            stages {
                stage('BE · Instalar dependencias') {
                    steps {
                        dir('back-canipet-api') {
                            sh 'pnpm install --frozen-lockfile'
                        }
                    }
                }
                stage('BE · Lint') {
                    steps {
                        dir('back-canipet-api') {
                            sh 'pnpm run lint --max-warnings 0 || true'
                        }
                    }
                }
                stage('BE · Tests unitarios') {
                    steps {
                        dir('back-canipet-api') {
                            sh 'pnpm run test --passWithNoTests'
                        }
                    }
                    post {
                        always {
                            junit(
                                testResults: 'back-canipet-api/coverage/junit.xml',
                                allowEmptyResults: true
                            )
                        }
                    }
                }
                stage('BE · Build') {
                    steps {
                        dir('back-canipet-api') {
                            sh 'pnpm run build'
                        }
                    }
                }
            }
        }

        // ── 2. Frontend ─────────────────────────────────────────────────────
        stage('Frontend') {
            stages {
                stage('FE · Instalar dependencias') {
                    steps {
                        dir('front-canipet-web') {
                            sh 'pnpm install --frozen-lockfile'
                        }
                    }
                }
                stage('FE · Lint') {
                    steps {
                        dir('front-canipet-web') {
                            sh 'pnpm run lint || true'
                        }
                    }
                }
                stage('FE · Build') {
                    environment {
                        NEXT_PUBLIC_API_URL = 'http://localhost:3000/api'
                    }
                    steps {
                        dir('front-canipet-web') {
                            sh 'pnpm run build'
                        }
                    }
                }
            }
        }

        // ── 3. Docker ───────────────────────────────────────────────────────
        stage('Docker · Construir imágenes') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                sh 'docker compose build --no-cache'
            }
        }

        // ── 4. Deploy ───────────────────────────────────────────────────────
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    docker compose down --remove-orphans || true
                    docker compose up -d --wait
                '''
            }
        }
    }

    // ── Post-pipeline ───────────────────────────────────────────────────────
    post {
        success {
            echo "✅ Pipeline exitoso en la rama ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline falló. Revisa los logs de la etapa fallida."
        }
        always {
            // Limpiar caché de builds intermedios de Next.js
            sh 'rm -rf front-canipet-web/.next/cache || true'
            cleanWs(
                cleanWhenSuccess: false,
                cleanWhenFailure: false,
                cleanWhenAborted: true
            )
        }
    }
}
