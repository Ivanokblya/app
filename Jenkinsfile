pipeline {
    agent any

    environment {
        STACK_NAME = "my_stack"
        COMPOSE_FILE = "docker-compose.yaml"
        DB_NAME = "lena"
        DB_USER = "myuser"
        DB_PASS = "mypassword"
        DB_HOST = "192.168.0.2"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Ivanokblya/app'
            }
        }
stage('Load Database from Git') {
    steps {
        sh """
            docker exec -i \$(docker ps -qf "name=db") mysql -u root -psecret lena < lena_dump.sql
        """
    }
}

        stage('Check Database Structure') {
            steps {
                script {
                    echo "Проверка наличия столбца first_name в таблице clients..."

                    def columnName = sh(
                        script: """mysql --skip-ssl -h ${DB_HOST} -u ${DB_USER} -p${DB_PASS} -D ${DB_NAME} -N -e "SHOW COLUMNS FROM clients LIKE 'first%';" | awk '{print \$1}'""",
                        returnStdout: true
                    ).trim()

                    echo "Обнаружено имя столбца: '${columnName}'"

                    if (columnName == "first_name") {
                        echo "Структура таблицы корректна. Продолжаем деплой."
                    } else {
                        error("Ошибка: неверное имя столбца ('${columnName}'). Ожидалось 'first_name'. Деплой остановлен.")
                    }
                }
            }
        }

        stage('Deploy to Swarm') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker stack deploy -c ${COMPOSE_FILE} ${STACK_NAME}
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
