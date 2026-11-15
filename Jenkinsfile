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

                    annuler() {
                        echo "Une nouvelle copie n est pas en bonne sante : annulation"
                        NOUVEAUX=$(docker ps -aq --filter label=com.docker.compose.project=sunusante --filter label=com.docker.compose.service=backend | grep -v -F "$ANCIENS")
                        for C in $NOUVEAUX; do
                            echo "===== Diagnostic de la copie $C ====="
                            docker inspect -f 'Etat={{.State.Status}} OOMKilled={{.State.OOMKilled}} CodeSortie={{.State.ExitCode}} Redemarrages={{.RestartCount}}' $C || true
                            docker logs --tail 60 $C 2>&1 || true
                            docker network disconnect sunusante_default $C || true
                        done
                        sleep 12
                        [ -n "$NOUVEAUX" ] && docker rm -f $NOUVEAUX
                        echo "Les anciennes copies continuent de servir"
                        exit 1
                    }

                    echo "1. Demarrage de la premiere nouvelle copie"
                    docker compose up -d --no-deps --no-recreate --scale backend=$((NB + 1)) --wait --wait-timeout 150 backend || annuler

                    echo "2. Demarrage de la deuxieme nouvelle copie"
                    docker compose up -d --no-deps --no-recreate --scale backend=$((NB + 2)) --wait --wait-timeout 150 backend || annuler

                    echo "Nouvelles copies en bonne sante. Attente que Nginx les repere"
                    sleep 15

                    echo "3. Retrait des anciennes copies de l annuaire"
                    for C in $ANCIENS; do
                        docker network disconnect sunusante_default $C
                    done

                    echo "Attente que Nginx oublie les anciennes adresses"
                    sleep 12

                    echo "4. Arret des anciennes copies"
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