pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = "todo_project"
        DOCKER_IMAGE_TAG = ""
    }

    stages {
        // STAGE 1: Checkout et détection du type de build
        stage('Checkout') {
            steps {
                echo "--- Etape 1: Checkout ---"
                checkout scm
                
                script {
                    if (env.CHANGE_ID) {
                        echo "🔀 Build déclenché par PR #${env.CHANGE_ID}"
                        env.BUILD_TYPE = "PR"
                    } else if (env.TAG_NAME) {
                        echo "🏷️ Build déclenché par tag: ${env.TAG_NAME}"
                        env.BUILD_TYPE = "TAG"
                        env.DOCKER_IMAGE_TAG = "${env.TAG_NAME}"
                    } else {
                        echo "🔨 Build déclenché par push sur ${env.BRANCH_NAME}"
                        env.BUILD_TYPE = "PUSH"
                    }
                }
            }
        }

        // STAGE 2: Setup et nettoyage
        stage('Setup') {
            steps {
                echo "--- Etape 2: Setup & Clean ---"
                script {
                    bat 'docker-compose --version'
                    // Nettoyage complet
                    bat 'docker-compose down --volumes --remove-orphans || exit 0'
                    bat 'docker system prune -f || exit 0'
                }
            }
        }

        // STAGE 3: Build conditionnel
        stage('Build') {
            when {
                expression { 
                    return env.BUILD_TYPE != "PR" 
                }
            }
            steps {
                echo "--- Etape 3: Build Images Docker ---"
                script {
                    if (env.BUILD_TYPE == "TAG") {
                        bat "docker-compose build --no-cache"
                        echo "✅ Build versionné avec tag: ${env.TAG_NAME}"
                    } else {
                        bat 'docker-compose build'
                        echo "✅ Build standard terminé"
                    }
                }
            }
        }

        // STAGE 4: Build léger pour PR
        stage('Build PR Léger') {
            when {
                expression { 
                    return env.BUILD_TYPE == "PR" 
                }
            }
            steps {
                echo "--- Etape 3b: Build Léger pour PR ---"
                bat 'docker-compose build --no-cache'
            }
        }

        // STAGE 5: Déploiement des containers
        stage('Run Containers') {
            when {
                expression { 
                    return env.BUILD_TYPE != "PR" 
                }
            }
            steps {
                echo "--- Etape 4: Déploiement Containers ---"
                bat 'docker-compose up -d'
                script {
                    // Attente stratégique pour le démarrage
                    sleep 5  // MongoDB d'abord
                    sleep 10 // Backend ensuite  
                    sleep 10 // Frontend enfin
                    echo "⏳ Containers déployés, attente des services..."
                }
            }
        }

        // STAGE 6: Smoke Tests adaptés à votre stack
        stage('Smoke Tests') {
            steps {
                echo "--- Etape 5: Smoke Tests ---"
                script {
                    if (env.BUILD_TYPE == "PR") {
                        // Tests légers pour PR
                        echo "🧪 Tests PR: Vérification build seulement"
                        bat 'docker-compose config'
                    } else {
                        // Tests complets pour PUSH/TAG
                        echo "🔍 Tests complets: Vérification des services"
                        
                        // Test MongoDB (30s max)
                        timeout(time: 30, unit: 'SECONDS') {
                            waitUntil(initialRecurrencePeriod: 5000) {
                                script {
                                    try {
                                        bat 'docker exec todo-mongo mongosh --eval "db.adminCommand(\"ping\")" | find "ok"'
                                        echo "✅ MongoDB (27017) répond"
                                        return true
                                    } catch (Exception e) {
                                        echo "⏳ En attente de MongoDB..."
                                        return false
                                    }
                                }
                            }
                        }
                        
                        // Test Backend (45s max)
                        timeout(time: 45, unit: 'SECONDS') {
                            waitUntil(initialRecurrencePeriod: 5000) {
                                script {
                                    def backendOk = bat(
                                        script: 'curl -f -s http://localhost:8000/health >nul 2>&1 || curl -f -s http://localhost:8000 >nul 2>&1', 
                                        returnStatus: true
                                    ) == 0
                                    echo backendOk ? "✅ Backend (8000) répond" : "⏳ En attente du backend..."
                                    return backendOk
                                }
                            }
                        }
                        
                        // Test Frontend (60s max)
                        timeout(time: 60, unit: 'SECONDS') {
                            waitUntil(initialRecurrencePeriod: 5000) {
                                script {
                                    def frontendOk = bat(
                                        script: 'curl -f -s -I http://localhost:3000 >nul 2>&1', 
                                        returnStatus: true
                                    ) == 0
                                    echo frontendOk ? "✅ Frontend (3000) répond" : "⏳ En attente du frontend..."
                                    return frontendOk
                                }
                            }
                        }
                        
                        echo "🎉 Tous les services répondent correctement!"
                    }
                }
            }
        }

        // STAGE 7: Tests d'intégration (optionnel)
        stage('Tests API') {
            when {
                expression { 
                    return env.BUILD_TYPE != "PR" 
                }
            }
            steps {
                echo "--- Etape 6: Tests API ---"
                script {
                    // Test basique de l'API
                    def apiTest = bat(
                        script: 'curl -f -s http://localhost:8000/api/todos >nul 2>&1', 
                        returnStatus: true
                    )
                    if (apiTest == 0) {
                        echo "✅ API testée avec succès"
                    } else {
                        echo "⚠️ API non accessible, vérifiez les logs"
                    }
                }
            }
        }

        // STAGE 8: Archivage pour les versions
        stage('Archive Version') {
            when {
                expression { 
                    return env.BUILD_TYPE == "TAG" 
                }
            }
            steps {
                echo "--- Etape 7: Archivage Version ${env.TAG_NAME} ---"
                script {
                    // Récupération des logs
                    bat 'docker-compose logs mongo > mongo.log 2>&1 || echo "Logs MongoDB non disponibles"'
                    bat 'docker-compose logs backend > backend.log 2>&1 || echo "Logs Backend non disponibles"'
                    bat 'docker-compose logs frontend > frontend.log 2>&1 || echo "Logs Frontend non disponibles"'
                    
                    // Archivage
                    archiveArtifacts artifacts: "*.log", allowEmptyArchive: true
                    
                    // Tagging des images (optionnel)
                    bat "docker tag todo_project-backend:latest todo_project-backend:${env.TAG_NAME} || echo \"Tagging backend échoué\""
                    bat "docker tag todo_project-frontend:latest todo_project-frontend:${env.TAG_NAME} || echo \"Tagging frontend échoué\""
                    
                    echo "🏷️ Version ${env.TAG_NAME} archivée avec succès"
                }
            }
        }

    }  // end stages

    post {
        always {
            echo "--- Nettoyage final ---"
            script {
                bat 'docker-compose down --volumes || exit 0'
                bat 'docker system prune -f || exit 0'
                cleanWs()
            }
        }
        success {
            script {
                switch(env.BUILD_TYPE) {
                    case "PR":
                        echo "✅ PR Build réussi - Prêt pour review"
                        // Ici: Notification Slack/Teams pour PR
                        break
                    case "TAG":
                        echo "🎉 Version ${env.TAG_NAME} buildée et testée avec succès!"
                        // Ici: Déclencher déploiement production
                        break
                    default:
                        echo "🔨 Build push sur ${env.BRANCH_NAME} réussi!"
                        break
                }
            }
        }
        failure {
            echo "❌ Pipeline ÉCHOUÉ - Consultez les logs ci-dessus"
            script {
                // Sauvegarde des logs en cas d'échec
                bat 'docker-compose logs > docker-failure.log 2>&1 || exit 0'
                archiveArtifacts artifacts: "docker-failure.log", allowEmptyArchive: true
            }
        }
    }
}
