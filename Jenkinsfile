// Pipeline de integración continua del gestor de tareas (Entrega semana 5).
// Funciona en Windows (bat) y en Linux/macOS (sh): la función ejecutar() elige sola.

def ejecutar(String comando) {
  if (isUnix()) {
    sh comando
  } else {
    bat comando
  }
}

pipeline {
  agent any

  // Jenkins revisa GitHub cada ~2 minutos y construye solo si hay commits nuevos.
  triggers {
    pollSCM('H/2 * * * *')
  }

  options {
    timestamps()
    timeout(time: 15, unit: 'MINUTES')
    disableConcurrentBuilds()   // dos construcciones a la vez chocarían en el mismo puerto
  }

  environment {
    COMPOSE_PROJECT_NAME = 'gestor-tareas-ci'
    APP_PORT = '3099'
    DB_USER = 'tareas'
    DB_NAME = 'tareas'
    DB_PASSWORD = credentials('tareas-db-password')
    ETAPA = 'inicio'
  }

  stages {
    stage('Obtener codigo') {
      steps {
        script { env.ETAPA = 'Obtener codigo' }
        checkout scm
      }
    }

    stage('Construir imagen') {
      steps {
        script {
          env.ETAPA = 'Construir imagen'
          ejecutar('docker compose build')
        }
      }
    }

    stage('Levantar contenedores') {
      steps {
        script {
          env.ETAPA = 'Levantar contenedores'
          // --wait espera a que ambos contenedores estén "healthy".
          ejecutar('docker compose up -d --wait')
          ejecutar('docker compose ps')
        }
      }
    }

    stage('Operacion de ejemplo') {
      steps {
        script {
          env.ETAPA = 'Operacion de ejemplo'
          ejecutar('docker compose exec -T app node scripts/operacion-ejemplo.js')
        }
      }
    }
  }

  post {
    failure {
      echo "La construccion FALLO en la etapa: ${env.ETAPA}"
      script {
        ejecutar('docker compose logs --no-color --tail=50')
      }
    }
    success {
      echo 'Construccion exitosa: la operacion de ejemplo dio el resultado esperado.'
    }
    always {
      script {
        ejecutar('docker compose down -v --remove-orphans')
      }
    }
  }
}
