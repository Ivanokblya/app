pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: tools
    image: alpine/k8s:1.27.4
    command:
    - cat
    tty: true
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
  - name: mysql-client
    image: mysql:8.0
    command:
    - cat
    tty: true
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
  - name: jnlp
    image: jenkins/inbound-agent:latest
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    resources:
      requests:
        cpu: "50m"
        memory: "256Mi"
"""
        }
    }

    environment {
        KUBECONFIG = credentials('kubeconfig-secret-id')
    }

    stages {
        stage('Check Kubernetes') {
            steps {
                container('tools') {
                    sh 'kubectl get nodes'
                    sh 'kubectl get namespaces'
                }
            }
        }

        stage('Test Database Connection') {
            steps {
                container('tools') {
                    script {
                        echo "Проверка доступности базы данных..."
                        def dbService = "db.default.svc.cluster.local"
                        echo "Попытка подключения к: ${dbService}:3306"

                        def dnsStatus = sh(script: "nslookup ${dbService}", returnStatus: true)
                        if (dnsStatus != 0) error("DNS ошибка — база недоступна!")

                        def endpointCheck = sh(script: "kubectl get endpoints db -n default -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null", returnStdout: true).trim()
                        if (endpointCheck == "") error("Нет endpoints — база недоступна!")

                        def podStatus = sh(script: "kubectl get pods -l app=mysql -n default -o jsonpath='{.items[0].status.phase}' 2>/dev/null", returnStdout: true).trim()
                        if (podStatus != 'Running') error("Pod базы не Running!")

                        def checkCmd = "nc -z -w 5 ${dbService} 3306"
                        def dbStatus = sh(script: checkCmd, returnStatus: true)
                        if (dbStatus != 0) error("База недоступна по TCP! Попробуйте проверить вручную в контейнере tools")

                        echo "База данных доступна по адресу: ${endpointCheck}:3306"
                    }
                }
            }
        }

        stage('Test Frontend') {
            steps {
                container('tools') {
                    script {
                        echo "Проверка доступности фронтенда..."

                        def serviceIP = sh(script: "kubectl get service crudback-service -n default -o jsonpath='{.spec.clusterIP}'", returnStdout: true).trim()
                        if (serviceIP == "") error("Сервис crudback-service не найден!")

                        def externalIP = sh(script: "kubectl get service crudback-service -n default -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true).trim()

                        if (externalIP) {
                            echo "External IP: ${externalIP}"
                            def frontendStatus = sh(script: "curl -sSf http://${externalIP}:80 -m 10 -o /dev/null", returnStatus: true)
                            if (frontendStatus != 0) error("Фронтенд недоступен по External IP")
                        } else {
                            echo "External IP отсутствует — проверяем через ClusterIP"
                            def port = sh(script: "kubectl get svc crudback-service -n default -o jsonpath='{.spec.ports[0].port}'", returnStdout: true).trim()
                            def internal = sh(script: "curl -sSf http://${serviceIP}:${port} -m 10 -o /dev/null", returnStatus: true)
                            if (internal != 0) error("Фронтенд недоступен через ClusterIP")
                        }

                        echo "Фронтенд доступен"
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                container('tools') {
                    script {
                        echo "Деплой приложения..."

                        sh 'kubectl set image deployment/crudback-app crudback=ivashka3228/crudback300 -n default'
                        sh 'kubectl rollout status deployment/crudback-app -n default --timeout=120s'

                        def selector = sh(script: 'kubectl get deployment crudback-app -n default -o jsonpath="{.spec.selector.matchLabels.app}"', returnStdout: true).trim()
                        echo "Selector: ${selector}"

                        def podCount = sh(script: "kubectl get pods -n default -l app=${selector} --no-headers | wc -l", returnStdout: true).trim()
                        if (podCount.toInteger() == 0) error("Поды не запущены!")

                        echo "Поды найдены: ${podCount}"
                        sh "kubectl get pods -n default -l app=${selector} -o wide"
                    }
                }
            }
        }

        stage('Validate Order Dates') {
            steps {
                container('mysql-client') {
                    script {
                        echo "Проверка корректности дат в таблице orders..."

                        def mysqlCmd = """
                            mysql -h db.default.svc.cluster.local -u root -p'secret' lena -e "
                            SELECT COUNT(*) FROM orders WHERE order_date > NOW()
                            OR order_date < '2000-01-01'
                            OR order_date IS NULL;"
                        """

                        def result = sh(script: mysqlCmd, returnStdout: true).trim()
                        def lines = result.split('\\n')
                        def count = lines.length > 1 ? lines[1].trim().toInteger() : 0

                        if (count > 0) {
                            error("Найдены некорректные даты в orders: ${count} записей")
                        } else {
                            echo "Все даты корректны"
                        }
                    }
                }
            }
        }

        stage('Final Health Check') {
            steps {
                container('tools') {
                    script {
                        echo "Финальная проверка..."

                        def selector = sh(script: 'kubectl get deployment crudback-app -n default -o jsonpath="{.spec.selector.matchLabels.app}"', returnStdout: true).trim()
                        def readyCount = sh(script: "kubectl get pods -n default -l app=${selector} -o jsonpath='{.items[*].status.containerStatuses[*].ready}' | grep -o true | wc -l", returnStdout: true).trim()
                        def totalCount = sh(script: "kubectl get pods -n default -l app=${selector} --no-headers | wc -l", returnStdout: true).trim()

                        echo "Готовность: ${readyCount}/${totalCount}"

                        if (readyCount != totalCount) error("Не все поды готовы!")

                        echo "Все поды готовы к работе"
                    }
                }
            }
        }

        stage('Diagnostics (before pod deletion)') {
            steps {
                container('tools') {
                    echo "Сбор диагностики..."
                    sh 'kubectl get all -n default || true'
                    sh 'kubectl get events -n default --sort-by=".lastTimestamp" | tail -10 || true'
                }
            }
        }
    }

    post {
        always {
            echo "Пайплайн завершён"
        }
        success {
            echo "Деплой успешно завершён!"
        }
        failure {
            echo "Деплой завершился с ошибками"
        }
        unstable {
            echo "Деплой завершился с предупреждениями"
        }
    }
}
