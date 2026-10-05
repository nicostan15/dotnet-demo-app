pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker compose build'
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
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test TodoApp.Tests/TodoApp.Tests.csproj

                    docker cp . "$TEST_CONTAINER":/src

                    docker start -a "$TEST_CONTAINER"

                    docker rm "$TEST_CONTAINER"
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
}