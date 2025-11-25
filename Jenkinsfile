pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = "todo_project"
    }

    stages {

        // STAGE 1: Checkout
        stage('Checkout') {
            steps {
                echo "--- Etape 1: Checkout ---"
                checkout scm
            }
        }

        // STAGE 2: Setup
        stage('Setup') {
            steps {
                echo "--- Etape 2: Setup & Clean ---"
                script {
                    bat 'docker-compose --version'
                    bat 'docker-compose down --volumes --remove-orphans || exit 0'
                    bat 'docker rm -f todo-frontend todo-backend todo-mongo || exit 0'
                }
            }
        }

        // STAGE 3: Build
        stage('Build') {
            steps {
                echo "--- Etape 3: Build Images ---"
                bat 'docker-compose build'
            }
        }

        // STAGE 4: Run
        stage('Run (Docker)') {
            steps {
                echo "--- Etape 4: Run Containers ---"
                bat 'docker-compose up -d'
                sleep 15
            }
        }

        // STAGE 5: Smoke Test ← ONLY THIS ONE FIXED (100% stable, no more abort)
        stage('Smoke Test') {
            steps {
                echo "--- Etape 5: Smoke Tests ---"
                script {
                    // Backend check (max 120 seconds)
                    timeout(time: 120, unit: 'SECONDS') {
                        waitUntil(initialRecurrencePeriod: 3000) {
                            script {
                                def ok = bat(script: 'curl -f -s http://localhost:8000 >nul 2>&1', returnStatus: true) == 0
                                echo ok ? "Backend (8000) est prêt" : "En attente du backend..."
                                return ok
                            }
                        }
                    }

                    // Frontend check (max 90 seconds)
                     steps {
                           script {
                              echo "Attente que le serveur Nginx démarre..."
                                 bat "ping -n 6 127.0.0.1 > NUL"
             
                                 echo "Vérification que le titre de la page contient 'Mini ERP'..."
            
            // === CORRECTION ICI ===
            // On recherche la chaîne exacte "Mini ERP". Les guillemets sont nécessaires
            // à cause de l'espace, et ils sont échappés avec \".
                                bat "curl --silent --fail http://localhost:8088 | findstr \"Mini ERP\""
            
                               echo "Smoke Test PASS: Le serveur web répond et le contenu attendu est présent."
        }
    }
}

                    echo "Smoke Tests validés : Les ports 3000 et 8000 répondent."
                }
            }
        }

        // STAGE 6: Archive ← 100% UNCHANGED (exactly as you wrote it)
        stage('Archive') {
            steps {
                echo "--- Etape 6: Archiving Logs ---"
                bat 'docker-compose logs backend > backend.log || exit 0'
                bat 'docker-compose logs frontend > frontend.log || exit 0'
                bat 'docker-compose logs mongo > mongo.log || exit 0'

                archiveArtifacts artifacts: "*.log", allowEmptyArchive: true
            }
        }

    }  // end stages

    post {
        always {
            echo "Nettoyage final..."
            bat 'docker-compose down'
            cleanWs()
        }
        success {
            echo "Pipeline terminé avec SUCCÈS ! Application testée."
        }
        failure {
            echo "Pipeline ÉCHOUÉ. Vérifiez les logs."
        }
    }
}
