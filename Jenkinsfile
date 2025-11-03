pipeline {
    agent any

    environment {
        STACK_NAME = "my_stack"
        COMPOSE_FILE = "docker-compose.yaml"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Ivanokblya/app'
            }
        }

        stage('Deploy to Swarm') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
                    docker stack deploy -c ${COMPOSE_FILE} ${STACK_NAME}
                    """
                }
            }
        }
         stage('Check DB columns') {
    steps {
        script {
            def mysqlHost = "localhost"
            def mysqlUser = "root"
            def mysqlPass = "secret"
            def mysqlDB = "lena"

            def result = sh(
                script: """mysql -h ${mysqlHost} -u ${mysqlUser} -p${mysqlPass} -D ${mysqlDB} -N -e "SHOW COLUMNS FROM clients LIKE 'first_name';" """,
                returnStdout: true
            ).trim()

            def resultHyphen = sh(
                script: """mysql -h ${mysqlHost} -u ${mysqlUser} -p${mysqlPass} -D ${mysqlDB} -N -e "SHOW COLUMNS FROM clients LIKE 'first-name';" """,
                returnStdout: true
            ).trim()

            echo "Check first_name column: '${result}'"
            echo "Check first-name column: '${resultHyphen}'"

            if (result == "") {
                error("Column 'first_name' not found in table 'clients'!")
            }
            if (resultHyphen != "") {
                error("Column 'first-name' should NOT exist!")
            }
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
