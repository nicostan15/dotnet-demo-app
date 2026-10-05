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
                    docker run --rm \
                      -v "$WORKSPACE":/src \
                      -w /src \
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test dotnet-demo.slnx
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