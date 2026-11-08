pipeline {
    agent any

    environment {
        DB_PASSWORD = credentials('db-password')
        COMPOSE_PROJECT_NAME = 'sunusante'
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
                    for i in $(seq 1 12); do
                        STATUT=$(docker inspect -f '{{.State.Health.Status}}' sunusante-db-1)
                        echo "Etat de la base : $STATUT"
                        if [ "$STATUT" = "healthy" ]; then
                            exit 0
                        fi
                        sleep 5
                    done
                    echo "La base n est pas en bonne sante apres 60 secondes"
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Deploiement de SunuSante reussi'
        }
        failure {
            echo 'Echec du deploiement de SunuSante'
        }
    }
}