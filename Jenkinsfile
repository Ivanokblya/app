pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: tools
                    image: alpine/k8s:1.27.4
                    command: ['cat']
                    tty: true
                    resources:
                      requests:
                        cpu: "100m"
                        memory: "128Mi"
                  - name: jnlp
                    image: jenkins/inbound-agent:latest
                    args: ['$(JENKINS_SECRET)', '$(JENKINS_NAME)']
                    resources:
                      requests:
                        cpu: "50m"
                        memory: "256Mi"
            '''
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
                        def mysqlAddress = "mysql-master.default.svc.cluster.local"
                        echo "Попытка подключения к: ${mysqlAddress}:3306"

                        def dnsStatus = sh(script: "nslookup ${mysqlAddress}", returnStatus: true)
                        if (dnsStatus != 0) error("DNS ошибка — база недоступна!")

                        def endpointCheck = sh(script: "kubectl get endpoints mysql-master -n default -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null", returnStdout: true).trim()
                        if (endpointCheck == "") error("Нет endpoints — база недоступна!")

                        def podStatus = sh(script: "kubectl get pods -l app=mysql-master -n default -o jsonpath='{.items[0].status.phase}' 2>/dev/null", returnStdout: true).trim()
                        if (podStatus != 'Running') error("Pod базы не Running!")

                        def dbStatus = sh(script: "timeout 10 sh -c 'echo > /dev/tcp/${mysqlAddress}/3306' 2>/dev/null", returnStatus: true)
                        if (dbStatus != 0) error("База недоступна по TCP!")

                        echo "База данных доступна: ${endpointCheck}:3306"
                    }
                }
            }
        }

        stage('Test Frontend') {
            steps {
                container('tools') {
                    script {
                        echo "Проверка доступности фронтенда..."

                        def serviceCheck = sh(script: "kubectl get service crudback-service -n default -o jsonpath='{.spec.clusterIP}'", returnStdout: true).trim()
                        if (serviceCheck == "") error("Сервис crudback-service не найден!")

                        def externalIP = sh(script: "kubectl get service crudback-service -n default -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true).trim()

                        if (externalIP) {
                            echo "External IP: ${externalIP}"
                            def frontendStatus = sh(script: "curl -sSf http://${externalIP}:80 -m 10 -o /dev/null", returnStatus: true)
                            if (frontendStatus != 0) error("Фронтенд недоступен по External IP")
                        } else {
                            echo "External IP нет — проверяем через ClusterIP"
                            def port = sh(script: "kubectl get svc crudback-service -n default -o jsonpath='{.spec.ports[0].port}'", returnStdout: true).trim()
                            def internal = sh(script: "curl -sSf http://${serviceCheck}:${port} -m 10 -o /dev/null", returnStatus: true)
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

                        // --- ЗАМЕНЁН ОБРАЗ ---
                        sh 'kubectl set image deployment/crudback-app crudback=ivashka3228/crudback -n default'

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

        stage('Final Health Check') {
            steps {
                container('tools') {
                    script {
                        echo "Финальная проверка..."

                        def selector = sh(script: 'kubectl get deployment crudback-app -n default -o jsonpath="{.spec.selector.matchLabels.app}"', returnStdout: true).trim()

                        def ready = sh(script: "kubectl get pods -n default -l app=${selector} -o jsonpath='{.items[*].status.containerStatuses[*].ready}' | grep -o true | wc -l", returnStdout: true).trim()
                        def total = sh(script: "kubectl get pods -n default -l app=${selector} --no-headers | wc -l", returnStdout: true).trim()

                        echo "Готовность: ${ready}/${total}"

                        if (ready != total) error("Не все поды готовы!")

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
