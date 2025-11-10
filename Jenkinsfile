pipeline {
    agent any

    environment {
        STACK_NAME = "my_stack"
        COMPOSE_FILE = "docker-compose.yaml"
        DB_NAME = "lena"
        DB_USER = "myuser"
        DB_PASS = "mypassword"
        DB_ROOT_PASS = "secret"
        DB_CONTAINER = "db"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Ivanokblya/app'
            }
        }

        stage('Deploy Docker Stack') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo "Deploying stack..."
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker stack deploy -c ${COMPOSE_FILE} ${STACK_NAME}
                        echo "Waiting for MySQL to start..."
                        sleep 20
                    """
                }
            }
        }

        stage('Load Database from Git') {
            steps {
                script {
                    echo "Импорт дампа базы данных из lena_dump.sql..."
                    sh """
                        DB_CONTAINER_ID=\$(docker ps -qf "name=${DB_CONTAINER}")
                        if [ -z "\$DB_CONTAINER_ID" ]; then
                            echo "Контейнер базы данных не найден!"
                            exit 1
                        fi
                        cat lena_dump.sql | docker exec -i \$DB_CONTAINER_ID mysql -u root -p${DB_ROOT_PASS} ${DB_NAME}
                    """
                }
            }
        }

        stage('Check Database Structure') {
    steps {
        script {
            echo "Проверка наличия столбца first_name в таблице clients..."

            // Находим ID контейнера базы данных по имени сервиса
            def dbContainer = sh(
                script: "docker ps -qf 'name=my_stack_db'",
                returnStdout: true
            ).trim()

            if (!dbContainer) {
                error("Не найден контейнер базы данных (my_stack_db). Проверь деплой.")
            }

            // Проверяем имя столбца в таблице
            def columnName = sh(
                script: """docker exec ${dbContainer} mysql -u root -psecret -D lena -N -e "SHOW COLUMNS FROM clients LIKE 'first%';" | awk '{print \$1}'""",
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


        stage('Post-Deploy Info') {
            steps {
                echo 'Deployment completed successfully!'
            }
        }
    }

    post {
        failure {
            echo 'Deployment failed!'
        }
    }
}
