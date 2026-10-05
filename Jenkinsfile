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
                        break
                      fi

                      echo "Waiting for MariaDB test database ($i/20)..."
                      sleep 3
                    done

                    if ! docker exec todoapp-test-db \
                      mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                      echo "MariaDB test database did not become ready"
                      docker logs todoapp-test-db
                      exit 1
                    fi
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

                    docker exec todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "SHOW TABLES FROM todo_test_db;"
                '''
            }
        }

        stage('Verify database from network') {
            steps {
                sh '''
                    set -e

                    docker run --rm \
                      --network ci-test-network \
                      mariadb:11 \
                      mariadb \
                        -h todoapp-test-db \
                        -P 3306 \
                        -utodo_usr \
                        -pletmeinplz \
                        todo_test_db \
                        -e "SELECT 1 AS database_connection_ok;"
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    TEST_CONTAINER=dotnet-test-runner

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker create \
                      --name "$TEST_CONTAINER" \
                      --network ci-test-network \
                      -w /src \
                      -e ConnectionStrings__TodoDb="Server=todoapp-test-db;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
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
                sh '''
                    docker rm -f todoapp-test-db || true
                    docker network rm ci-test-network || true
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    docker compose down || true
                    docker compose up -d

                    for i in $(seq 1 20); do
                      if docker exec todoappdb \
                        mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                        echo "Deployment database is ready"
                        break
                      fi

                      echo "Waiting for deployment database ($i/20)..."
                      sleep 3
                    done

                    if ! docker exec todoappdb \
                      mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                      echo "Deployment database did not become ready"
                      docker logs todoappdb
                      exit 1
                    fi

                    docker exec -i todoappdb \
                      mariadb -h 127.0.0.1 -uroot -psekrit todo_db \
                      < TodoApp/schema.sql

                    docker exec todoappdb \
                      mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "SHOW TABLES FROM todo_db;"

                    docker compose restart todoapp
                    docker compose ps
                '''
            }
        }

        stage('Acceptance test') {
            steps {
                sh '''
                    set -e

                    for i in $(seq 1 12); do
                      if docker run --rm \
                        --network dotnettodopipeline_default \
                        curlimages/curl:8.12.1 \
                        -fsS http://todoapp:8080/ > /tmp/todoapp.html; then

                        echo "Todo-app is reachable from the Docker network"
                        cat /tmp/todoapp.html
                        exit 0
                      fi

                      echo "Waiting for todo-app ($i/12)..."
                      sleep 5
                    done

                    echo "Todo-app did not become reachable from the Docker network"
                    docker logs todoapp || true
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
        }
    }
}