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
                echo '=== ANALIZANDO src/ CON OWASP ==='
                sh '''
                    mkdir -p reports/dependency-check

                    /opt/dependency-check/bin/dependency-check.sh \
                      --project "clindata-app-src" \
                      --scan src \
                      --format HTML \
                      --out reports/dependency-check \
                      --enableRetired \
                      --log reports/dependency-check/owasp-sca.log
		      --data /var/owasp-data

                    echo "✅ Reporte generado en reports/dependency-check"
                '''
            }
        }
    }

    post {
        always {
            echo '''
            =========================================
            RESUMEN DE ANÁLISIS CON DEPENDENCY-CHECK
            =========================================
            ✅ Reporte generado: revisar resultados
            '''

            archiveArtifacts artifacts: 'reports/dependency-check/dependency-check-report.html', onlyIfSuccessful: true
        }

        success {
            echo '🎉 ANÁLISIS COMPLETADO EXITOSAMENTE'
        }

        failure {
            echo '❌ ERROR DURANTE EL ANÁLISIS - REVISAR LOGS'
        }
    }
}
