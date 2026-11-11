
pipeline {
    agent any

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "================================"
                    echo "Building branch: $BRANCH_NAME"
                    echo "================================"

                    node -v
                    npm -v
                '''
            }
        }

        stage('Configure Port') {
            steps {
                script {
                    def ports = [
                        dev       : '3001',
                        staging   : '3002',
                        main      : '3003',
                        production: '3004'
                    ]

                    if (!ports.containsKey(env.BRANCH_NAME)) {
                        error("Unsupported branch: ${env.BRANCH_NAME}")
                    }

                    env.PORT = ports[env.BRANCH_NAME]
                    env.VITE_BRANCH = env.BRANCH_NAME
                    env.PID_FILE = "/tmp/react-${env.BRANCH_NAME}.pid"
                }

                echo "Branch: ${env.BRANCH_NAME}"
                echo "Port: ${env.PORT}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build React') {
            steps {
                sh '''
                    echo "Building React application..."

                    npm run build
                '''
            }
        }

        stage('Stop Previous Application') {
            steps {
                sh '''
                    if [ -f "$PID_FILE" ]; then
                        OLD_PID=$(cat "$PID_FILE")

                        if kill -0 "$OLD_PID" 2>/dev/null; then
                            echo "Stopping previous application: $OLD_PID"
                            kill "$OLD_PID" || true
                            sleep 2
                        fi

                        rm -f "$PID_FILE"
                    fi
                '''
            }
        }

        stage('Run React Application') {
            steps {
                sh '''
                    echo "Starting React application on port $PORT"

                    JENKINS_NODE_COOKIE=dontKillMe nohup npm run preview -- \
                        --host 0.0.0.0 \
                        --port "$PORT" \
                        --strictPort \
                        > "$WORKSPACE/react-server.log" 2>&1 &

                    echo $! > "$PID_FILE"

                    sleep 5

                    if ! curl -fsS "http://127.0.0.1:$PORT" > /dev/null; then
                        echo "Application failed to start!"
                        cat "$WORKSPACE/react-server.log"
                        exit 1
                    fi

                    echo "Application started successfully!"
                    echo "URL: http://SERVER-IP:$PORT"
                '''
            }
        }
    }

    post {
        success {
            echo """
                ====================================
                BUILD SUCCESSFUL
                Branch: ${env.BRANCH_NAME}
                Port: ${env.PORT}
                ====================================
            """
        }

        failure {
            echo "Pipeline failed for branch: ${env.BRANCH_NAME}"
        }
    }
}
