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

                    docker network create ci-test-network 2>/dev/null || true
                    docker rm -f todoapp-test-db 2>/dev/null || true

                    docker run -d \
                      --name todoapp-test-db \
                      --network ci-test-network \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_test_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      mariadb:11

                    for i in $(seq 1 20); do
                      if docker exec todoapp-test-db \
                        mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                        echo "Test database is ready"
                        exit 0
                      fi

                      echo "Waiting for MariaDB test database ($i/20)..."
                      sleep 3
                    done

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
                      mariadb -h 127.0.0.1 -uroot -psekrit todo_test_db \
                      < TodoApp/schema.sql
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    cat > "$WORKSPACE/test-appsettings.json" <<'EOF'
{
  "ConnectionStrings": {
    "TodoDb": "Server=todoapp-test-db;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;"
  }
}
EOF

                    docker run --rm \
                      --name dotnet-test-runner \
                      --network ci-test-network \
                      -w /src/TodoApp.Tests \
                      -v "$WORKSPACE":/src:ro \
                      -v "$WORKSPACE/test-appsettings.json":/src/TodoApp.Tests/appsettings.json:ro \
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test TodoApp.Tests.csproj
                '''
            }
        }

        stage('Cleanup test database') {
            steps {
                sh '''
                    docker rm -f todoapp-test-db || true
                    docker network rm ci-test-network || true
                '''
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
            sh 'docker network rm ci-test-network 2>/dev/null || true'
            sh 'rm -f "$WORKSPACE/test-appsettings.json" 2>/dev/null || true'
        }
    }
}