// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
//
// Esta es la versión BASE: construye, prueba y despliega en producción.
// Los marcadores  >>> [ETIQUETA]  indican dónde agregar las etapas de seguridad.
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push (tuyo o un PR de Dependabot fusionado) dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Security Review (SAST)') {
            steps {
                script {
                    // Reportes: JSON completo (incluye Low) para la trazabilidad y txt legible
                    sh 'bandit -f json -o reports/bandit.json app.py || true'
                    sh 'bandit -f txt -o reports/bandit.txt app.py || true'
                    // UNSTABLE (no FAIL): el build vulnerable debe terminar y archivar TODOS los reportes
                    def hallazgos = sh(script: 'bandit -ll app.py', returnStatus: true)
                    if (hallazgos != 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Bandit: hallazgos Medium/High -> build UNSTABLE'
                    } else {
                        echo 'Bandit: sin hallazgos Medium/High'
                    }
                }
            }
        }

        stage('Dependency Management (SCA)') {
            steps {
                script {
                    sh 'pip-audit -r requirements.txt -f json --progress-spinner off -o reports/pip-audit.json || true'
                    sh 'pip-audit -r requirements.txt -f columns --progress-spinner off -o reports/pip-audit.txt || true'
                    def vulnerabilidades = sh(script: 'pip-audit -r requirements.txt --progress-spinner off', returnStatus: true)
                    if (vulnerabilidades != 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'pip-audit: vulnerabilidades conocidas -> build UNSTABLE'
                    } else {
                        echo 'pip-audit: sin vulnerabilidades conocidas'
                    }
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    // zap_scan.py termina con 2 si hay alertas High/Medium
                    def zap = sh(script: 'python3 scripts/zap_scan.py http://jenkins:5001', returnStatus: true)
                    if (zap == 2) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'ZAP: alertas High/Medium -> build UNSTABLE'
                    } else if (zap != 0) {
                        error("ZAP terminó con código ${zap}")
                    } else {
                        echo 'ZAP: sin alertas High/Medium'
                    }
                }
            }
        }

        stage('Deploy Producción') {
            // Solo se despliega si no hubo hallazgos de seguridad: Bandit, pip-audit y ZAP en verde
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
