pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: tools
                    image: bitnami/kubectl:latest
                    command:
                    - cat
                    tty: true
                    resources:
                      requests:
                        cpu: "100m"
                        memory: "128Mi"
                    volumeMounts:
                    - mountPath: "/home/jenkins/agent"
                      name: "workspace-volume"
                      readOnly: false
                  - name: jnlp
                    image: jenkins/inbound-agent:latest
                    args: ['$(JENKINS_SECRET)', '$(JENKINS_NAME)']
                    resources:
                      requests:
                        cpu: "50m"
                        memory: "256Mi"
                    volumeMounts:
                    - mountPath: "/home/jenkins/agent"
                      name: "workspace-volume"
                      readOnly: false
                volumes:
                - emptyDir: {}
                  name: "workspace-volume"
                nodeSelector:
                  kubernetes.io/os: linux
                restartPolicy: Never
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
                        def dbService = "db.default.svc.cluster.local"
                        echo "Попытка подключения к: ${dbService}:3306"

                        def dnsStatus = sh(script: "nslookup ${dbService}", returnStatus: true)
                        if (dnsStatus != 0) error("DNS ошибка — база недоступна!")

                        def endpointCheck = sh(script: "kubectl get endpoints db -n default -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null", returnStdout: true).trim()
                        if (endpointCheck == "") error("Нет endpoints — база недоступна!")

                        def podStatus = sh(script: "kubectl get pods -l app=mysql -n default -o jsonpath='{.items[0].status.phase}' 2>/dev/null", returnStdout: true).trim()
                        if (podStatus != 'Running') error("Pod базы не Running!")

                        // Используем nc для проверки порта
                        def dbStatus = sh(script: "nc -z -w 10 ${dbService} 3306", returnStatus: true)
                        if (dbStatus != 0) error("База недоступна по TCP!")

                        echo "База данных доступна по адресу: ${endpointCheck}:3306"
                    }
                }
            }
        }

        // ... остальные стадии без изменений
    }

    // ... post-блок без изменений
}
