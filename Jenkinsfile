pipeline {
    agent any

    parameters {
        string(name: 'BACKEND_TAG', defaultValue: 'latest', description: 'Version de l image backend a deployer')
        string(name: 'FRONTEND_TAG', defaultValue: 'latest', description: 'Version de l image frontend a deployer')
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
        stage('Base de donnees') {
            steps {
                sh 'docker compose up -d --no-deps --wait db'
            }
        }
        stage('Backend sans coupure') {
            steps {
                sh '''
                    ANCIENS=$(docker ps -q --filter label=com.docker.compose.project=sunusante --filter label=com.docker.compose.service=backend)

                    if [ -z "$ANCIENS" ]; then
                        echo "Aucun backend en service : premier deploiement"
                        docker compose up -d --no-deps --wait --wait-timeout 150 backend
                        exit 0
                    fi

                    NB=$(echo "$ANCIENS" | wc -l)
                    echo "Copies actuelles du backend : $NB"

                    echo "1. Demarrage de 2 nouvelles copies a cote des anciennes"
                    if ! docker compose up -d --no-deps --no-recreate --scale backend=$((NB + 2)) --wait --wait-timeout 150 backend; then
                        echo "Les nouvelles copies ne sont pas en bonne sante : annulation"
                        NOUVEAUX=$(docker ps -aq --filter label=com.docker.compose.project=sunusante --filter label=com.docker.compose.service=backend | grep -v -F "$ANCIENS")
                        [ -n "$NOUVEAUX" ] && docker rm -f $NOUVEAUX
                        echo "Les anciennes copies continuent de servir"
                        exit 1
                    fi

                    echo "2. Nouvelles copies en bonne sante. Attente que Nginx les repere"
                    sleep 15

                    echo "3. Arret des anciennes copies"
                    docker stop $ANCIENS
                    docker rm $ANCIENS
                '''
            }
        }
        stage('Frontend') {
            steps {
                sh 'docker compose up -d --no-deps --wait frontend'
            }
        }
        stage('Verifier de bout en bout') {
            steps {
                sh 'docker compose ps'
                sh 'docker compose exec -T frontend wget -qO- http://127.0.0.1/api/etablissements > /dev/null'
                echo 'Frontend, backend et base repondent'
            }
        }
    }

    post {
        success {
            echo "Deploiement de SunuSante reussi (backend ${params.BACKEND_TAG}, frontend ${params.FRONTEND_TAG})"
        }
        failure {
            echo 'Echec du deploiement de SunuSante'
        }
    }
}