pipeline {
  agent any
  options {
    timestamps()
    parallelsAlwaysFailFast()   // si una rama paralela falla, cancela las demás
  }

  environment {
    // Config común
    DB_HOST = 'localhost'
    DB_PORT = '3306'
    APP_PROFILE = ''      // 'prod' si manejas perfiles de Spring; vacío si no

    // Puertos por ambiente (evitan choque local)
    DEV_PORT  = '8081'
    QA_PORT   = '8082'
    PROD_PORT = '8083'

    // DBs por ambiente (cámbialas si quieres)
    DEV_DB  = 'blog_dev'
    QA_DB   = 'blog_qa'
    PROD_DB = 'blog_prod'

    // Credenciales de ejemplo (cámbialas en tu entorno real)
    DEV_USER  = 'root'
    DEV_PASS  = 'root'
    QA_USER   = 'root'
    QA_PASS   = 'root'
    PROD_USER = 'root'
    PROD_PASS = 'root'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'master', url: 'https://github.com/Jorge-Urquiza/blog-back.git'
      }
    }

    stage('Build (sin tests)') {
      steps {
        bat '.\\mvnw.cmd -B clean package -Dmaven.test.skip=true'
      }
    }

    stage('Prepare (artefacto & git)') {
      steps {
        script {
          // Detectar JAR construido
          def listOut = bat(returnStdout: true, script: '@echo off\r\ndir /b target\\*.jar').trim()
          def jarLines = listOut.readLines()
          if (!jarLines || jarLines.isEmpty()) {
            error 'No se encontró ningún JAR en target/*.jar'
          }
          env.JAR_PATH = ("target\\" + jarLines[0].trim())

          // SHA corto para nombrar artefacto
          def shaOut = bat(returnStdout: true, script: '@echo off\r\ngit rev-parse --short=12 HEAD').trim()
          def lines = shaOut.readLines()
          env.GIT_SHA = lines[lines.size() - 1].trim()

          env.BUILD_NAME = "blog-back-${env.BUILD_NUMBER}-${env.GIT_SHA}.jar"
        }
      }
    }

    stage('Stamp & Archive') {
      steps {
        bat """
          copy /Y "${env.JAR_PATH}" "target\\${env.BUILD_NAME}"
          certutil -hashfile "target\\${env.BUILD_NAME}" SHA256 > "target\\${env.BUILD_NAME}.sha256.txt"
        """
        archiveArtifacts artifacts: "target/${env.BUILD_NAME}, target/${env.BUILD_NAME}.sha256.txt", fingerprint: true
      }
    }

    // --------- Despliegues no productivos en paralelo ---------
    stage('Deploy non-prod (parallel)') {
      parallel {
        stage('Deploy DEV (smoke)') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
              timeout(time: 40, unit: 'SECONDS') {
                withEnv([
                  "SPRING_PROFILES_ACTIVE=${APP_PROFILE}",
                  "SPRING_DATASOURCE_URL=jdbc:mysql://${DB_HOST}:${DB_PORT}/${DEV_DB}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC",
                  "SPRING_DATASOURCE_USERNAME=${DEV_USER}",
                  "SPRING_DATASOURCE_PASSWORD=${DEV_PASS}",
                  "SPRING_JPA_HIBERNATE_DDL_AUTO=update",
                  "SERVER_PORT=${DEV_PORT}"
                ]) {
                  bat """
                    echo [DEV] Ejecutando: ${env.JAR_PATH} en puerto %SERVER_PORT% con DB ${DEV_DB}
                    java -jar "${env.JAR_PATH}"
                  """
                }
              }
            }
          }
        }

        stage('Deploy QA (smoke)') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
              timeout(time: 40, unit: 'SECONDS') {
                withEnv([
                  "SPRING_PROFILES_ACTIVE=${APP_PROFILE}",
                  "SPRING_DATASOURCE_URL=jdbc:mysql://${DB_HOST}:${DB_PORT}/${QA_DB}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC",
                  "SPRING_DATASOURCE_USERNAME=${QA_USER}",
                  "SPRING_DATASOURCE_PASSWORD=${QA_PASS}",
                  "SPRING_JPA_HIBERNATE_DDL_AUTO=update",
                  "SERVER_PORT=${QA_PORT}"
                ]) {
                  bat """
                    echo [QA] Ejecutando: ${env.JAR_PATH} en puerto %SERVER_PORT% con DB ${QA_DB}
                    java -jar "${env.JAR_PATH}"
                  """
                }
              }
            }
          }
        }
      }
    }

    // Gate manual antes de producción
    stage('Aprobación para PRODUCCIÓN') {
      steps {
        input message: "¿Deploy a PRODUCCIÓN con artefacto ${env.BUILD_NAME}?", ok: "Continuar"
      }
    }

    stage('Deploy PROD (smoke)') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          timeout(time: 40, unit: 'SECONDS') {
            withEnv([
              "SPRING_PROFILES_ACTIVE=${APP_PROFILE}",
              "SPRING_DATASOURCE_URL=jdbc:mysql://${DB_HOST}:${DB_PORT}/${PROD_DB}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC",
              "SPRING_DATASOURCE_USERNAME=${PROD_USER}",
              "SPRING_DATASOURCE_PASSWORD=${PROD_PASS}",
              "SPRING_JPA_HIBERNATE_DDL_AUTO=update",
              "SERVER_PORT=${PROD_PORT}"
            ]) {
              bat """
                echo [PROD] Ejecutando: ${env.JAR_PATH} en puerto %SERVER_PORT% con DB ${PROD_DB}
                java -jar "${env.JAR_PATH}"
              """
            }
          }
        }
      }
    }
  }

  post {
    always {
      echo "Artefactos publicados en 'Artifacts'. Los deploys son *smoke* con timeout (UNSTABLE esperado)."
    }
  }
}
