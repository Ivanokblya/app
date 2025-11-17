pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: tools
                    image: lachlanevenson/k8s-kubectl:latest
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
                    sh '/bin/bash -c "kubectl get nodes"'
                    sh '/bin/bash -c "kubectl get namespaces"'
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

                        def dnsStatus = sh(script: "/bin/bash -c 'nslookup ${dbService}'", returnStatus: true)
                        if (dnsStatus != 0) error("DNS ошибка — база недоступна!")

                        def endpointCheck = sh(script: "/bin/bash -c \"kubectl get endpoints db -n default -o jsonpath='{.subsets[0].addresses[0].ip}'\"", returnStdout: true).trim()
                        if (endpointCheck == "") error("Нет endpoints — база недоступна!")

                        def podStatus = sh(script: "/bin/bash -c \"kubectl get pods -l app=mysql -n default -o jsonpath='{.items[0].status.phase}'\"", returnStdout: true).trim()
                        if (podStatus != 'Running') error("Pod базы не Running!")

                        def dbStatus = sh(script: "/bin/bash -c 'nc -z -w 10 ${dbService} 3306'", returnStatus: true)
                        if (dbStatus != 0) error("База недоступна по TCP!")

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

                        def serviceIP = sh(script: "/bin/bash -c \"kubectl get service crudback-service -n default -o jsonpath='{.spec.clusterIP}'\"", returnStdout: true).trim()
                        if (serviceIP == "") error("Сервис crudback-service не найден!")

                        def externalIP = sh(script: "/bin/bash -c \"kubectl get service crudback-service -n default -o jsonpath='{.status.loadBalancer.ingress[0].ip}'\"", returnStdout: true).trim()

                        if (externalIP) {
                            echo "External IP: ${externalIP}"
                            def frontendStatus = sh(script: "/bin/bash -c 'curl -sSf http://${externalIP}:80 -m 10 -o /dev/null'", returnStatus: true)
                            if (frontendStatus != 0) error("Фронтенд недоступен по External IP")
                        } else {
                            echo "External IP отсутствует — проверяем через ClusterIP"
                            def port = sh(script: "/bin/bash -c \"kubectl get svc crudback-service -n default -o jsonpath='{.spec.ports[0].port}'\"", returnStdout: true).trim()
                            def internal = sh(script: "/bin/bash -c 'curl -sSf http://${serviceIP}:${port} -m 10 -o /dev/null'", returnStatus: true)
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

                        sh '/bin/bash -c "kubectl set image deployment/crudback-app crudback=ivashka3228/crudback -n default"'
                        sh '/bin/bash -c "kubectl rollout status deployment/crudback-app -n default --timeout=120s"'

                        def selector = sh(script: "/bin/bash -c 'kubectl get deployment crudback-app -n default -o jsonpath=\"{.spec.selector.matchLabels.app}\"'", returnStdout: true).trim()
                        echo "Selector: ${selector}"

                        def podCount = sh(script: "/bin/bash -c \"kubectl get pods -n default -l app=${selector} --no-headers | wc -l\"", returnStdout: true).trim()
                        if (podCount.toInteger() == 0) error("Поды не запущены!")

                        echo "Поды найдены: ${podCount}"
                        sh "/bin/bash -c \"kubectl get pods -n default -l app=${selector} -o wide\""
                    }
                }
            }
        }

        stage('Final Health Check') {
            steps {
                container('tools') {
                    script {
                        echo "Финальная проверка..."

                        def selector = sh(script: "/bin/bash -c 'kubectl get deployment crudback-app -n default -o jsonpath=\"{.spec.selector.matchLabels.app}\"'", returnStdout: true).trim()
                        def readyCount = sh(script: "/bin/bash -c \"kubectl get pods -n default -l app=${selector} -o jsonpath='{.items[*].status.containerStatuses[*].ready}' | grep -o true | wc -l\"", returnStdout: true).trim()
                        def totalCount = sh(script: "/bin/bash -c \"kubectl get pods -n default -l app=${selector} --no-headers | wc -l\"", returnStdout: true).trim()

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
                    sh '/bin/bash -c "kubectl get all -n default || true"'
                    sh '/bin/bash -c "kubectl get events -n default --sort-by=.lastTimestamp | tail -10 || true"'
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
