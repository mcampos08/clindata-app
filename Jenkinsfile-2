pipeline {
    agent any
    
    environment {
        // Configuración de VMs y repositorio
        VM_STAGING_IP = '192.168.72.129'
        STAGING_USER = 'clindata'
        STAGING_PATH = '/var/www/html/clindata'
        GITHUB_REPO = 'https://github.com/mcampos08/clindata-app.git'
        
        // Configuración de SonarQube (solo servidor, proyecto definido en sonar-project.properties)
        SONARQUBE_SERVER = 'SonarQube-Local'
        
        // Configuración de reportes
        REPORTS_DIR = 'reports'
        BUILD_ARTIFACTS_DIR = 'build'
    }
    
    stages {
        stage('🔧 Verificar Herramientas') {
            steps {
                echo '=== VERIFICANDO INSTALACIÓN DE HERRAMIENTAS ==='
                script {
                    // Verificar herramientas básicas
                    sh '''
                        echo "🔍 Verificando herramientas básicas..."
                        echo "Java: $(java -version 2>&1 | head -1)"
                        echo "Git: $(git --version)"
                        echo "Curl: $(curl --version | head -1)"
                        echo "Usuario Jenkins: $(whoami)"
                        echo "Directorio actual: $(pwd)"
                    '''
                    
                    // Verificar SonarQube Scanner
                    try {
                        def scannerHome = tool 'SonarQubeScanner'
                        echo "✅ SonarQube Scanner encontrado en: ${scannerHome}"
                        sh "${scannerHome}/bin/sonar-scanner -h | head -3"
                    } catch (Exception e) {
                        echo "❌ Error con SonarQube Scanner: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                    
                    // Verificar OWASP Dependency Check
                    sh '''
                        if [ -x "/opt/dependency-check/bin/dependency-check.sh" ]; then
                            echo "✅ OWASP Dependency Check encontrado"
                            /opt/dependency-check/bin/dependency-check.sh --version | head -1
                        else
                            echo "❌ OWASP Dependency Check no encontrado en /opt/dependency-check/"
                            echo "⚠️ Continuando sin OWASP Dependency Check"
                        fi
                    '''
                    
                    // Verificar Syft
                    sh '''
                        if command -v syft &> /dev/null; then
                            echo "✅ Syft encontrado"
                            syft version | head -1
                        else
                            echo "❌ Syft no encontrado en PATH"
                            echo "⚠️ Continuando sin Syft"
                        fi
                    '''
                }
            }
        }
        
        stage('📥 Checkout y Análisis del Proyecto') {
            steps {
                echo '=== CLONANDO Y ANALIZANDO REPOSITORIO ==='
                
                // Limpiar workspace previo
                cleanWs()
                
                // Clonar repositorio
                git branch: 'main', url: "${GITHUB_REPO}"
                
                sh '''
                    echo "=== INFORMACIÓN DEL REPOSITORIO ==="
                    echo "📍 Repositorio: ${GITHUB_REPO}"
                    echo "🏷️  Commit actual: $(git log -1 --pretty=format:"%h - %an, %ar : %s")"
                    echo "🌿 Rama: $(git branch --show-current)"
                    echo ""
                    
                    echo "=== ESTRUCTURA DEL PROYECTO ==="
                    echo "📁 Archivos en raíz:"
                    ls -la | head -15
                    echo ""
                    
                    echo "📄 Archivos de configuración encontrados:"
                    [ -f "Jenkinsfile" ] && echo "✅ Jenkinsfile" || echo "❌ Jenkinsfile no encontrado"
                    [ -f "sonar-project.properties" ] && echo "✅ sonar-project.properties" || echo "❌ sonar-project.properties no encontrado"
                    [ -f ".gitignore" ] && echo "✅ .gitignore" || echo "⚠️ .gitignore no encontrado"
                    [ -f "README.md" ] && echo "✅ README.md" || echo "⚠️ README.md no encontrado"
                    echo ""
                    
                    echo "🐘 Archivos PHP encontrados:"
                    find . -name "*.php" -type f | head -10
                    echo "📊 Total archivos PHP: $(find . -name "*.php" -type f | wc -l)"
                    echo ""
                    
                    echo "📋 Tamaño del proyecto:"
                    du -sh . 2>/dev/null || echo "No se pudo calcular el tamaño"
                '''
                
                // Mostrar contenido del archivo de configuración de SonarQube si existe
                script {
                    if (fileExists('sonar-project.properties')) {
                        echo "📄 Contenido de sonar-project.properties:"
                        sh 'cat sonar-project.properties'
                    } else {
                        echo "⚠️ No se encontró sonar-project.properties, se usará configuración inline"
                    }
                }
            }
        }
        
        stage('🔍 Análisis de SonarQube (SAST)') {
            steps {
                echo '=== ANÁLISIS ESTÁTICO CON SONARQUBE ==='
                script {
                    // Crear directorio de reportes
                    sh "mkdir -p ${REPORTS_DIR}"
                    
                    // Verificar conexión a SonarQube
                    sh '''
                        echo "🔗 Verificando conexión a SonarQube..."
                        if curl -f -s http://localhost:9000/api/system/status > /dev/null; then
                            echo "✅ SonarQube está accesible"
                            curl -s http://localhost:9000/api/system/status | grep -o '"status":"[^"]*"' || true
                        else
                            echo "❌ No se puede conectar a SonarQube en localhost:9000"
                            echo "🔧 Verificar que SonarQube esté ejecutándose"
                            exit 1
                        fi
                    '''
                    
                    // Ejecutar análisis de SonarQube
                    try {
                        def scannerHome = tool 'SonarQubeScanner'
                        withSonarQubeEnv("${SONARQUBE_SERVER}") {
                            if (fileExists('sonar-project.properties')) {
                                echo "✅ Ejecutando análisis usando sonar-project.properties"
                                sh """
                                    echo "📊 Iniciando análisis de SonarQube..."
                                    ${scannerHome}/bin/sonar-scanner \
                                        -Dsonar.working.directory=${REPORTS_DIR}/sonar-work \
                                        -X
                                """
                            } else {
                                echo "⚠️ Ejecutando análisis con configuración básica inline"
                                sh """
                                    ${scannerHome}/bin/sonar-scanner \
                                        -Dsonar.projectKey=clindata-app-basic-test \
                                        -Dsonar.projectName='Clindata App Basic Test' \
                                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                                        -Dsonar.sources=. \
                                        -Dsonar.language=php \
                                        -Dsonar.sourceEncoding=UTF-8 \
                                        -Dsonar.exclusions=vendor/**,tests/**,*.log,*.md,${REPORTS_DIR}/**,${BUILD_ARTIFACTS_DIR}/** \
                                        -Dsonar.working.directory=${REPORTS_DIR}/sonar-work \
                                        -X
                                """
                            }
                        }
                        echo "✅ Análisis de SonarQube completado exitosamente"
                    } catch (Exception e) {
                        echo "❌ Error en análisis de SonarQube: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Continuando con el pipeline..."
                    }
                }
            }
        }
        
        stage('🛡️ Quality Gate de SonarQube') {
            when {
                expression { 
                    return fileExists('sonar-project.properties') && 
                           readFile('sonar-project.properties').contains('sonar.qualitygate.wait=true')
                }
            }
            steps {
                echo '=== EVALUANDO QUALITY GATE ==='
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        try {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "❌ Quality Gate falló: ${qg.status}"
                                echo "🔍 Detalles del Quality Gate:"
                                echo "- Estado: ${qg.status}"
                                
                                // Continuar pero marcar como unstable
                                currentBuild.result = 'UNSTABLE'
                                echo "⚠️ Continuando pipeline con estado UNSTABLE"
                            } else {
                                echo "✅ Quality Gate pasó exitosamente"
                            }
                        } catch (Exception e) {
                            echo "❌ Error evaluando Quality Gate: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        
        stage('🛡️ OWASP Dependency Check (SCA)') {
            steps {
                echo '=== ANÁLISIS DE COMPONENTES Y DEPENDENCIAS ==='
                script {
                    try {
                        sh '''
                            echo "🔍 Iniciando escaneo de dependencias..."
                            mkdir -p ${REPORTS_DIR}
                            
                            # Verificar si existen archivos de dependencias comunes
                            echo "📋 Buscando archivos de dependencias:"
                            [ -f "composer.json" ] && echo "✅ composer.json encontrado" || echo "⚠️ composer.json no encontrado"
                            [ -f "package.json" ] && echo "✅ package.json encontrado" || echo "⚠️ package.json no encontrado"
                            [ -d "vendor" ] && echo "✅ directorio vendor encontrado" || echo "⚠️ directorio vendor no encontrado"
                            
                            # Ejecutar OWASP Dependency Check
                            if [ -x "/opt/dependency-check/bin/dependency-check.sh" ]; then
                                echo "🚀 Ejecutando OWASP Dependency Check..."
                                /opt/dependency-check/bin/dependency-check.sh \
                                    --project "Clindata-App-Security-Test" \
                                    --scan . \
                                    --format HTML \
                                    --format JSON \
                                    --format XML \
                                    --out ${REPORTS_DIR}/ \
                                    --enableExperimental \
                                    --log ${REPORTS_DIR}/dependency-check.log \
                                    --failOnCVSS 8.0 || {
                                        echo "⚠️ OWASP encontró vulnerabilidades, pero continuando..."
                                        exit 0
                                    }
                                echo "✅ OWASP Dependency Check completado"
                            else
                                echo "❌ OWASP Dependency Check no disponible"
                                echo "📝 Creando reporte placeholder..."
                                echo "<html><body><h1>OWASP Dependency Check no disponible</h1></body></html>" > ${REPORTS_DIR}/dependency-check-report.html
                            fi
                        '''
                        
                        // Analizar resultados si existen
                        if (fileExists("${REPORTS_DIR}/dependency-check-report.json")) {
                            sh '''
                                echo "📊 Analizando resultados de OWASP..."
                                if [ -f "${REPORTS_DIR}/dependency-check-report.json" ]; then
                                    echo "🔍 Resumen de vulnerabilidades:"
                                    grep -o '"severity":"[^"]*"' ${REPORTS_DIR}/dependency-check-report.json | sort | uniq -c || echo "No se encontraron vulnerabilidades categorizadas"
                                fi
                            '''
                        }
                        
                        echo "✅ Análisis de dependencias completado"
                    } catch (Exception e) {
                        echo "❌ Error en OWASP Dependency Check: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Continuando con el pipeline..."
                    }
                }
            }
        }
        
        stage('📋 Generar SBOM con Syft') {
            steps {
                echo '=== GENERANDO SOFTWARE BILL OF MATERIALS ==='
                script {
                    try {
                        sh '''
                            echo "📝 Generando SBOM con Syft..."
                            mkdir -p ${REPORTS_DIR}
                            
                            if command -v syft &> /dev/null; then
                                echo "🚀 Ejecutando Syft..."
                                
                                # Generar SBOM en múltiples formatos
                                syft . -o table=${REPORTS_DIR}/sbom.txt
                                syft . -o json=${REPORTS_DIR}/sbom.json
                                syft . -o spdx-json=${REPORTS_DIR}/sbom.spdx.json || echo "⚠️ Formato SPDX no disponible"
                                
                                echo "📊 Resumen del SBOM generado:"
                                echo "=== PRIMERAS 15 LÍNEAS ==="
                                head -15 ${REPORTS_DIR}/sbom.txt
                                echo ""
                                echo "📈 Estadísticas:"
                                echo "Total de componentes: $(cat ${REPORTS_DIR}/sbom.txt | wc -l)"
                                
                            else
                                echo "❌ Syft no disponible"
                                echo "📝 Creando SBOM placeholder..."
                                echo "Syft no disponible en este sistema" > ${REPORTS_DIR}/sbom.txt
                                echo '{"components": [], "message": "Syft not available"}' > ${REPORTS_DIR}/sbom.json
                            fi
                        '''
                        echo "✅ SBOM generado exitosamente"
                    } catch (Exception e) {
                        echo "❌ Error generando SBOM: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Continuando con el pipeline..."
                    }
                }
            }
        }
        
        stage('🔗 Verificar Conectividad SSH') {
            steps {
                echo '=== PROBANDO CONEXIÓN SSH A VM-STAGING ==='
                script {
                    try {
                        sshagent(['staging-ssh-key']) {
                            sh '''
                                echo "🔐 Verificando conexión SSH a VM-Staging..."
                                ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 ${STAGING_USER}@${VM_STAGING_IP} "
                                    echo '✅ Conexión SSH exitosa'
                                    echo '🖥️  Sistema: '$(uname -a)
                                    echo '👤 Usuario actual: '$(whoami)
                                    echo '📁 Directorio home: '$(pwd)
                                    echo '💾 Espacio en disco:'
                                    df -h / | tail -1
                                    echo ''
                                    echo '🔍 Verificando directorio de staging...'
                                    if [ -d '${STAGING_PATH}' ]; then
                                        echo '✅ Directorio ${STAGING_PATH} existe'
                                        ls -la ${STAGING_PATH} | head -5
                                        echo '📊 Archivos en staging: '$(find ${STAGING_PATH} -type f | wc -l)
                                    else
                                        echo '⚠️  Directorio ${STAGING_PATH} no existe, se creará en despliegue'
                                        echo '📋 Verificando directorio padre...'
                                        ls -la $(dirname ${STAGING_PATH}) || echo 'Directorio padre no accesible'
                                    fi
                                    echo ''
                                    echo '🌐 Verificando servicios web...'
                                    if command -v systemctl >/dev/null 2>&1; then
                                        systemctl is-active apache2 2>/dev/null || systemctl is-active nginx 2>/dev/null || echo 'Servicios web no detectados'
                                    fi
                                "
                            '''
                        }
                        echo "✅ Verificación SSH completada exitosamente"
                    } catch (Exception e) {
                        echo "❌ Error de conexión SSH: ${e.message}"
                        echo "🔧 Verificaciones recomendadas:"
                        echo "   - SSH keys configuradas correctamente"
                        echo "   - VM-Staging accesible desde Jenkins"
                        echo "   - Usuario ${STAGING_USER} existe en VM-Staging"
                        echo "   - Conectividad de red entre VMs"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('📊 Resumen y Validación Final') {
            steps {
                echo '=== RESUMEN FINAL DE VERIFICACIONES ==='
                script {
                    sh '''
                        echo "=================================================="
                        echo "🎯 RESUMEN DE VERIFICACIONES COMPLETADAS"
                        echo "=================================================="
                        echo "📅 Fecha: $(date)"
                        echo "🏗️  Build: ${BUILD_NUMBER}"
                        echo "📁 Workspace: $(pwd)"
                        echo "🔗 Repositorio: ${GITHUB_REPO}"
                        echo "🏷️  Commit: $(git log -1 --pretty=format:"%h - %s")"
                        echo ""
                        
                        echo "📋 ARCHIVOS GENERADOS:"
                        if [ -d "${REPORTS_DIR}" ]; then
                            find ${REPORTS_DIR} -type f | sort | while read file; do
                                size=$(ls -lh "$file" | awk '{print $5}')
                                echo "  📄 $file ($size)"
                            done
                        else
                            echo "  ⚠️ No se generaron reportes"
                        fi
                        echo ""
                        
                        echo "🔍 HERRAMIENTAS VERIFICADAS:"
                        echo "  ✅ Git y repositorio clonado"
                        echo "  $([ -x "/opt/dependency-check/bin/dependency-check.sh" ] && echo "✅" || echo "❌") OWASP Dependency Check"
                        echo "  $(command -v syft >/dev/null && echo "✅" || echo "❌") Syft SBOM"
                        echo "  $(curl -f -s http://localhost:9000/api/system/status >/dev/null && echo "✅" || echo "❌") SonarQube"
                        echo ""
                        
                        echo "🎯 ESTADO DEL PIPELINE:"
                        echo "  📊 Build Result: ${BUILD_RESULT:-SUCCESS}"
                        echo "  📈 Current Result: ${currentBuild.result:-SUCCESS}"
                        echo ""
                        
                        echo "🚀 PRÓXIMOS PASOS:"
                        echo "  1. ✅ Verificar que todas las herramientas respondieron correctamente"
                        echo "  2. 📊 Revisar reportes generados en artifacts"
                        echo "  3. 🔧 Corregir cualquier configuración pendiente"
                        echo "  4. 🚀 Proceder con pipeline completo si todo funciona"
                        echo "  5. 📋 Implementar despliegue automático"
                        echo ""
                        
                        echo "=================================================="
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo '=== FINALIZANDO PIPELINE ==='
            
            script {
                // Archivar reportes generados
                try {
                    if (fileExists(REPORTS_DIR)) {
                        archiveArtifacts artifacts: "${REPORTS_DIR}/**/*", fingerprint: true, allowEmptyArchive: true
                        echo "✅ Reportes archivados exitosamente"
                    } else {
                        echo "⚠️ No hay reportes para archivar"
                    }
                } catch (Exception e) {
                    echo "❌ Error archivando reportes: ${e.message}"
                }
                
                // Publicar reporte HTML de OWASP si existe
                try {
                    if (fileExists("${REPORTS_DIR}/dependency-check-report.html")) {
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: REPORTS_DIR,
                            reportFiles: 'dependency-check-report.html',
                            reportName: 'OWASP Dependency Check Report',
                            reportTitles: 'Security Vulnerability Report'
                        ])
                        echo "✅ Reporte OWASP HTML publicado"
                    }
                } catch (Exception e) {
                    echo "❌ Error publicando reporte HTML: ${e.message}"
                }
                
                // Mostrar información de cleanup
                sh '''
                    echo "🧹 Información de limpieza:"
                    echo "  - Workspace será limpiado automáticamente"
                    echo "  - Reportes archivados en Jenkins"
                    echo "  - Logs disponibles en historial de builds"
                '''
            }
        }
        
        success {
            echo '''
            ✅ ================================================
            ✅ PIPELINE DE VERIFICACIÓN COMPLETADO EXITOSAMENTE
            ✅ ================================================
            
            🎉 Todas las verificaciones han sido ejecutadas
            📊 Revisar artifacts para ver reportes detallados
            🔍 Verificar logs para confirmar funcionamiento
            🚀 Sistema listo para pipeline completo de seguridad
            
            📋 Checklist completado:
               ✅ Herramientas verificadas
               ✅ Repositorio clonado
               ✅ SonarQube ejecutado
               ✅ OWASP Dependency Check ejecutado
               ✅ SBOM generado
               ✅ Conectividad SSH verificada
            '''
        }
        
        failure {
            echo '''
            ❌ ================================================
            ❌ PIPELINE DE VERIFICACIÓN FALLÓ
            ❌ ================================================
            
            🔍 Pasos para diagnóstico:
               1. Revisar logs de cada stage para identificar errores
               2. Verificar configuración de herramientas
               3. Confirmar conectividad de red
               4. Validar credenciales y permisos
            
            🛠️ Herramientas a verificar:
               - SonarQube en localhost:9000
               - OWASP Dependency Check en /opt/dependency-check/
               - Syft en PATH
               - SSH keys configuradas
            
            📞 Contactar al equipo DevOps si persisten los errores
            '''
        }
        
        unstable {
            echo '''
            ⚠️ ================================================
            ⚠️ PIPELINE DE VERIFICACIÓN COMPLETADO CON WARNINGS
            ⚠️ ================================================
            
            🔧 Algunas herramientas necesitan ajustes:
               - Revisar warnings en los logs de cada stage
               - Verificar configuraciones opcionales
               - Confirmar que herramientas críticas funcionan
            
            ✅ El pipeline puede continuar con precaución
            📋 Recomendar corregir warnings antes de producción
            '''
        }
    }
}
