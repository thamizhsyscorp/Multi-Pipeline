
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        APP_DIR = "${WORKSPACE}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "Current branch: $BRANCH_NAME"
                    node -v
                    npm -v
                '''
            }
        }

        stage('Configure Port') {
            steps {
                script {
                    def ports = [
                        dev: '3001',
                        staging: '3002',
                        main: '3003',
                        production: '3004'
                    ]

                    if (!ports.containsKey(env.BRANCH_NAME)) {
                        error("Unsupported branch: ${env.BRANCH_NAME}")
                    }

                    env.PORT = ports[env.BRANCH_NAME]
                    env.PID_FILE = "/tmp/react-${env.BRANCH_NAME}.pid"
                }
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
                            kill "$OLD_PID"
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
                    nohup npm run preview -- \
                        --host 0.0.0.0 \
                        --port "$PORT" \
                        --strictPort \
                        > "$WORKSPACE/react-server.log" 2>&1 &

                    echo $! > "$PID_FILE"

                    sleep 5

                    if ! curl -fsS "http://127.0.0.1:$PORT" > /dev/null; then
                        cat "$WORKSPACE/react-server.log"
                        exit 1
                    fi
                '''

                echo "React application running on port ${env.PORT}"
            }
        }
    }

    post {
        success {
            echo "Branch: ${env.BRANCH_NAME}"
            echo "Application running on port: ${env.PORT}"
        }

        failure {
            echo "Pipeline failed: ${env.BRANCH_NAME}"
        }
    }
}
