def deployOne(String label, String port) {
  bat """
    @echo off
    setlocal ENABLEDELAYEDEXPANSION

    echo [${label}] Arrancando app en puerto ${port} contra DB %DB_NAME%

    rem Construir URL JDBC SIN carets (^)
    set "JDBC_URL=jdbc:mysql://%DB_HOST%:%DB_PORT%/%DB_NAME%?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC"

    rem Lanzar la app en background
    start "" /B cmd /c ^
      java ^
        -Dspring.datasource.url="!JDBC_URL!" ^
        -Dspring.datasource.username=%DB_USER% ^
        -Dspring.datasource.password=%DB_PASS% ^
        -Dspring.jpa.hibernate.ddl-auto=update ^
        -Dserver.port=${port} ^
        -jar "${env.RELEASE_JAR}"

    set "PID="

    rem Espera hasta 120s a que el puerto esté LISTENING (1s por ciclo usando ping)
    for /L %%i in (1,1,120) do (
      ping -n 2 127.0.0.1 >nul
      for /f "tokens=5" %%a in ('netstat -ano ^| findstr ":${port} " ^| findstr LISTENING') do set "PID=%%a"
      if defined PID goto up_${label}
    )

    echo [${label}] No abrio el puerto ${port} a tiempo.
    rem Limpieza defensiva
    for /f "tokens=5" %%a in ('netstat -ano ^| findstr ":${port} " ^| findstr LISTENING') do taskkill /PID %%a /F >nul 2>&1
    endlocal
    exit /b 2

    :up_${label}
    echo [${label}] UP con PID !PID! en puerto ${port}. Apagando...
    taskkill /PID !PID! /F >nul 2>&1
    endlocal
    exit /b 0
  """
}

pipeline {
  agent any
  options { timestamps() }

  environment {
    // MISMA DB para DEV/QA/PROD
    DB_HOST = 'localhost'
    DB_PORT = '3306'
    DB_NAME = 'blog'
    DB_USER = 'user'
    DB_PASS = 'root'

    // Puertos distintos por ambiente
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

    stage('Build') {
      steps {
        bat '.\\mvnw.cmd -B clean package -Dmaven.test.skip=true'
      }
    }

    stage('Stamp & Archive') {
      steps {
        script {
          def listOut = bat(returnStdout: true, script: '@echo off\r\ndir /b target\\*.jar').trim()
          def jarLines = listOut.readLines()
          if (!jarLines || jarLines.isEmpty()) { error 'No se encontró ningún JAR en target/*.jar' }
          env.JAR_PATH = ("target\\" + jarLines[0].trim())

          def shaOut = bat(returnStdout: true, script: '@echo off\r\ngit rev-parse --short=12 HEAD').trim()
          def lines = shaOut.readLines()
          env.GIT_SHA = lines[lines.size() - 1].trim()

          env.BUILD_NAME  = "blog-back-${env.BUILD_NUMBER}-${env.GIT_SHA}.jar"
          env.RELEASE_JAR = "target\\${env.BUILD_NAME}"
        }

        bat """
          copy /Y "${env.JAR_PATH}" "${env.RELEASE_JAR}"
          certutil -hashfile "${env.RELEASE_JAR}" SHA256 > "${env.RELEASE_JAR}.sha256.txt"
        """
        archiveArtifacts artifacts: "target/${env.BUILD_NAME}, target/${env.BUILD_NAME}.sha256.txt", fingerprint: true
      }
    }

    stage('Deploy DEV/QA/PROD (parallel)') {
      parallel {
        stage('Deploy DEV')  { steps { script { deployOne('DEV',  env.DEV_PORT)  } } }
        stage('Deploy QA')   { steps { script { deployOne('QA',   env.QA_PORT)   } } }
        stage('Deploy PROD') { steps { script { deployOne('PROD', env.PROD_PORT) } } }
      }
    }
  }

  post {
    always {
      echo "Artefactos listos. DEV/QA/PROD verificados en paralelo contra la MISMA DB: ${env.DB_NAME}."
    }
  }
}

