
pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        PORT = ''
        APP_URL = ''
        PID_FILE = ''
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Configure Environment') {
            steps {
                script {
                    switch (env.BRANCH_NAME) {
                        case 'dev':
                            env.PORT = '3001'
                            break

                        case 'staging':
                            env.PORT = '3002'
                            break

                        case 'main':
                            env.PORT = '3003'
                            break

                        case 'production':
                            env.PORT = '3004'
                            break

                        default:
                            error("Unsupported branch: ${env.BRANCH_NAME}")
                    }

                    env.APP_URL = "http://localhost:${env.PORT}"
                    env.PID_FILE = "/tmp/react-${env.BRANCH_NAME}.pid"
                }

                echo "Application port: ${env.PORT}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test -- --run'
            }
        }

        stage('Build React') {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**',
                                 fingerprint: true
            }
        }

        stage('Production Approval') {
            when {
                branch 'production'
            }
            steps {
                input message: 'Approve production server startup?',
                      ok: 'Start Production'
            }
        }

        stage('Stop Previous Server') {
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

        stage('Start React Server') {
            steps {
                sh '''
                    JENKINS_NODE_COOKIE=dontKillMe \
                    nohup ./node_modules/.bin/vite preview \
                        --host 0.0.0.0 \
                        --port "$PORT" \
                        --strictPort \
                        > "react-server.log" 2>&1 &

                    echo $! > "$PID_FILE"

                    sleep 3

                    if ! kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
                        cat react-server.log
                        exit 1
                    fi
                '''

                echo "React server started on port ${env.PORT}"
            }
        }

        stage('Verify Application') {
            steps {
                sh '''
                    for i in $(seq 1 15); do
                        if curl -fsS "$APP_URL" > /dev/null; then
                            echo "React application is responding!"
                            exit 0
                        fi

                        sleep 2
                    done

                    cat react-server.log
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "Branch: ${env.BRANCH_NAME}"
            echo "Application: ${env.APP_URL}"
            echo "Pipeline completed successfully!"
        }

        failure {
            echo "Pipeline failed for ${env.BRANCH_NAME}"
        }
    }
}
