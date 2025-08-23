pipeline {
  agent any
  options {
    timestamps()
    // Si quieres que al fallar una rama se cancelen las demás, descomenta:
    // parallelsAlwaysFailFast()
  }

  environment {
    // DB única para los 3 ambientes
    DB_HOST = '127.0.0.1'
    DB_PORT = '3306'
    DB_NAME = 'blog'     // <<-- misma DB para DEV/QA/PROD
    DB_USER = 'root'
    DB_PASS = 'root'

    APP_PROFILE = ''     // usa 'prod' si manejas perfiles

    // Puertos por ambiente (distintos, pero misma DB)
    DEV_PORT  = '8081'
    QA_PORT   = '8082'
    PROD_PORT = '8083'
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

          // SHA corto para nombre
          def shaOut = bat(returnStdout: true, script: '@echo off\r\ngit rev-parse --short=12 HEAD').trim()
          def lines = shaOut.readLines()
          env.GIT_SHA = lines[lines.size() - 1].trim()

          env.BUILD_NAME  = "blog-back-${env.BUILD_NUMBER}-${env.GIT_SHA}.jar"
          env.RELEASE_JAR = "target\\${env.BUILD_NAME}"
        }
      }
    }

    stage('Stamp & Archive') {
      steps {
        bat """
          copy /Y "${env.JAR_PATH}" "${env.RELEASE_JAR}"
          certutil -hashfile "${env.RELEASE_JAR}" SHA256 > "${env.RELEASE_JAR}.sha256.txt"
        """
        archiveArtifacts artifacts: "target/${env.BUILD_NAME}, target/${env.BUILD_NAME}.sha256.txt", fingerprint: true
      }
    }

    // -------- Deploy paralelo: misma DB, distintos puertos --------
    stage('Deploy DEV/QA/PROD (parallel)') {
      parallel {
        stage('Deploy DEV (smoke)') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
              timeout(time: 40, unit: 'SECONDS') {
                withEnv(["ENV_PORT=${DEV_PORT}"]) {
                  bat """
                    echo [DEV] Ejecutando: ${env.RELEASE_JAR} en puerto %ENV_PORT% con DB %DB_NAME%
                    java -jar "${env.RELEASE_JAR}" ^
                      --spring.datasource.url="jdbc:mysql://%DB_HOST%:%DB_PORT%/%DB_NAME%?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" ^
                      --spring.datasource.username=%DB_USER% ^
                      --spring.datasource.password=%DB_PASS% ^
                      --spring.jpa.hibernate.ddl-auto=update ^
                      --server.port=%ENV_PORT%
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
                withEnv(["ENV_PORT=${QA_PORT}"]) {
                  bat """
                    echo [QA] Ejecutando: ${env.RELEASE_JAR} en puerto %ENV_PORT% con DB %DB_NAME%
                    java -jar "${env.RELEASE_JAR}" ^
                      --spring.datasource.url="jdbc:mysql://%DB_HOST%:%DB_PORT%/%DB_NAME%?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" ^
                      --spring.datasource.username=%DB_USER% ^
                      --spring.datasource.password=%DB_PASS% ^
                      --spring.jpa.hibernate.ddl-auto=update ^
                      --server.port=%ENV_PORT%
                  """
                }
              }
            }
          }
        }

        stage('Deploy PROD (smoke)') {
          steps {
            // PROD: si falla, que falle el pipeline
            timeout(time: 40, unit: 'SECONDS') {
              withEnv(["ENV_PORT=${PROD_PORT}"]) {
                bat """
                  echo [PROD] Ejecutando: ${env.RELEASE_JAR} en puerto %ENV_PORT% con DB %DB_NAME%
                  java -jar "${env.RELEASE_JAR}" ^
                    --spring.datasource.url="jdbc:mysql://%DB_HOST%:%DB_PORT%/%DB_NAME%?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" ^
                    --spring.datasource.username=%DB_USER% ^
                    --spring.datasource.password=%DB_PASS% ^
                    --spring.jpa.hibernate.ddl-auto=update ^
                    --server.port=%ENV_PORT%
                """
              }
            }
          }
        }
      }
    }
  }

  post {
    always {
      echo "Artefactos listos en 'Artifacts'. DEV/QA/PROD desplegados en paralelo usando la MISMA DB."
    }
  }
}
