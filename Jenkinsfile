pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Start test database') {
            steps {
                sh '''
                    set -e

                    docker rm -f todoapp-test-db 2>/dev/null || true

                    docker run -d \
                      --name todoapp-test-db \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_test_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      -p 3307:3306 \
                      mariadb:11

                    for i in $(seq 1 20); do
                      if docker exec todoapp-test-db \
                        mariadb-admin ping -h localhost -uroot -psekrit --silent; then
                        echo "Test database is ready"
                        exit 0
                      fi

                      echo "Waiting for MariaDB test database ($i/20)..."
                      sleep 3
                    done

                    echo "MariaDB test database did not become ready"
                    docker logs todoapp-test-db
                    exit 1
                '''
            }
        }

        stage('Initialize test database') {
            steps {
                sh '''
                    set -e

                    docker exec -i todoapp-test-db \
                      mariadb -uroot -psekrit todo_test_db \
                      < TodoApp/schema.sql
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    TEST_CONTAINER=dotnet-test-runner

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker create --name "$TEST_CONTAINER" \
                      -w /src \
                      --add-host=host.docker.internal:host-gateway \
                      -e ConnectionStrings__TodoDb="Server=host.docker.internal;Port=3307;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test TodoApp.Tests/TodoApp.Tests.csproj

                    docker cp . "$TEST_CONTAINER":/src

                    docker start -a "$TEST_CONTAINER"

                    docker rm "$TEST_CONTAINER"
                '''
            }
        }

        stage('Cleanup test database') {
            steps {
                sh 'docker rm -f todoapp-test-db || true'
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d'
                sh 'docker compose ps'
            }
        }

        stage('Acceptance test') {
            steps {
                sh '''
                    for i in $(seq 1 12); do
                      if curl -fsS http://localhost:8081/ > /tmp/todoapp.html; then
                        grep -q "<" /tmp/todoapp.html
                        exit 0
                      fi
                      echo "Waiting for todo-app ($i/12)..."
                      sleep 5
                    done

                    echo "Todo-app did not become reachable"
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f todoapp-test-db 2>/dev/null || true'
            sh 'docker rm -f dotnet-test-runner 2>/dev/null || true'
        }
    }
}