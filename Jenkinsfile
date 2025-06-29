pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/mcampos08/clindata-app.git'
    }

    stages {
        stage('📥 Checkout') {
            steps {
                echo '=== CLONANDO REPOSITORIO ==='
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('📦 Análisis SCA - OWASP Dependency Check') {
            steps {
                echo '=== ANALIZANDO composer.lock CON OWASP ==='
                script {
                    def dcHome = tool name: 'OWASP-Dependency-Check', type: 'org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation'
                    
                    sh """
                        mkdir -p reports/dependency-check

                        ${dcHome}/bin/dependency-check.sh \
                          --project "clindata-app-src" \
                          --scan src \
                          --format HTML \
                          --format JSON \
                          --out reports/dependency-check \
                          --enableRetired \
                          --data ${WORKSPACE}/.dependency-check-data \
                          --log reports/dependency-check/owasp-sca.log

                        echo "✅ Reporte generado en reports/dependency-check"
                    """

                    // Leer el JSON para imprimir advertencias si hay vulnerabilidades graves
                    def jsonReport = readJSON file: 'reports/dependency-check/dependency-check-report.json'
                    def highOrCritical = jsonReport.dependencies.findAll { dep ->
                        dep.vulnerabilities?.any { v -> v.severity in ['HIGH', 'CRITICAL'] }
                    }

                    if (highOrCritical && highOrCritical.size() > 0) {
                        echo "❌ Se detectaron ${highOrCritical.size()} dependencias con vulnerabilidades HIGH o CRITICAL"
                        echo "▶ Revisa el reporte HTML en la interfaz de Jenkins"
                    } else {
                        echo "✅ No se encontraron vulnerabilidades HIGH o CRITICAL"
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/dependency-check/dependency-check-report.html', onlyIfSuccessful: true

            publishHTML([
                reportName: 'Reporte OWASP Dependency-Check',
                reportDir: 'reports/dependency-check',
                reportFiles: 'dependency-check-report.html',
                keepAll: true,
                allowMissing: false,
                alwaysLinkToLastBuild: true
            ])
        }

        success {
            echo '🎉 ANÁLISIS DE DEPENDENCIAS COMPLETADO CON ÉXITO'
        }

        failure {
            echo '❌ FALLÓ EL ANÁLISIS DE DEPENDENCIAS - VERIFICAR LOGS'
        }
    }
}
