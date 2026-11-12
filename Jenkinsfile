pipeline {
    agent any

    parameters {
        string(name: 'BACKEND_TAG', defaultValue: 'latest', description: 'Version de l image backend a deployer')
    }

    environment {
        DB_PASSWORD = credentials('db-password')
        COMPOSE_PROJECT_NAME = 'sunusante'
    }
    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Verifier la configuration') {
            steps {
                sh 'docker compose config --quiet'
            }
        }
        stage('Recuperer les images') {
            steps {
                sh 'docker compose pull'
            }
        }
        stage('Deployer') {
            steps {
                sh 'docker compose up -d'
            }
        }
        stage('Verifier la sante') {
            steps {
                sh '''
                    for SERVICE in db backend; do
                        for i in $(seq 1 18); do
                            STATUT=$(docker inspect -f '{{.State.Health.Status}}' sunusante-$SERVICE-1)
                            echo "Etat de $SERVICE : $STATUT"
                            if [ "$STATUT" = "healthy" ]; then
                                break
                            fi
                            if [ "$i" -eq 18 ]; then
                                echo "$SERVICE n est pas en bonne sante apres 90 secondes"
                                exit 1
                            fi
                            sleep 5
                        done
                    done
                '''
            }
        }
    }

    post {
        success {
            echo "Deploiement de SunuSante reussi (backend version ${params.BACKEND_TAG})"
        }
        failure {
            echo 'Echec du deploiement de SunuSante'
        }
    }
}