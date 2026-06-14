pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        REGISTRY = 'registry.digitalocean.com/YOUR_REGISTRY'
        BACKEND_REPOSITORY = 'jrawler-backend'
        FRONTEND_REPOSITORY = 'jrawler-frontend'
        REGISTRY_CREDENTIALS_ID = 'digitalocean-registry'
        DEPLOY_SSH_CREDENTIALS_ID = 'digitalocean-droplet-ssh'
        DEPLOY_USER = 'root'
        DEPLOY_HOST = 'YOUR_DROPLET_IP'
        DEPLOY_DIR = '/opt/jrawler'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    env.BACKEND_IMAGE = "${env.REGISTRY}/${env.BACKEND_REPOSITORY}"
                    env.FRONTEND_IMAGE = "${env.REGISTRY}/${env.FRONTEND_REPOSITORY}"
                }
            }
        }

        stage('Backend Test') {
            steps {
                sh '''
                    mkdir -p .jenkins-m2
                    docker run --rm \
                        -v "$PWD/backend:/workspace" \
                        -v "$PWD/.jenkins-m2:/root/.m2" \
                        -w /workspace \
                        maven:3.9.9-eclipse-temurin-21 \
                        mvn -B test
                '''
            }
        }

        stage('Frontend Lint And Build') {
            steps {
                sh '''
                    mkdir -p .jenkins-npm
                    docker run --rm \
                        -v "$PWD/frontend:/workspace" \
                        -v "$PWD/.jenkins-npm:/root/.npm" \
                        -w /workspace \
                        node:22.13-alpine \
                        sh -lc "npm ci && npm run lint && npm run build"
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t "$BACKEND_IMAGE:$IMAGE_TAG" -t "$BACKEND_IMAGE:latest" backend
                    docker build -t "$FRONTEND_IMAGE:$IMAGE_TAG" -t "$FRONTEND_IMAGE:latest" frontend
                '''
            }
        }

        stage('Push Docker Images') {
            when {
                anyOf {
                    branch 'main'
                    expression { env.GIT_BRANCH == 'main' || env.GIT_BRANCH == 'origin/main' }
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.REGISTRY_CREDENTIALS_ID,
                    usernameVariable: 'REGISTRY_USERNAME',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh '''
                        echo "$REGISTRY_PASSWORD" | docker login "$REGISTRY" -u "$REGISTRY_USERNAME" --password-stdin
                        docker push "$BACKEND_IMAGE:$IMAGE_TAG"
                        docker push "$BACKEND_IMAGE:latest"
                        docker push "$FRONTEND_IMAGE:$IMAGE_TAG"
                        docker push "$FRONTEND_IMAGE:latest"
                    '''
                }
            }
        }

        stage('Deploy To DigitalOcean') {
            when {
                anyOf {
                    branch 'main'
                    expression { env.GIT_BRANCH == 'main' || env.GIT_BRANCH == 'origin/main' }
                }
            }
            steps {
                sshagent(credentials: [env.DEPLOY_SSH_CREDENTIALS_ID]) {
                    withCredentials([usernamePassword(
                        credentialsId: env.REGISTRY_CREDENTIALS_ID,
                        usernameVariable: 'REGISTRY_USERNAME',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    )]) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no "$DEPLOY_USER@$DEPLOY_HOST" "mkdir -p '$DEPLOY_DIR'"
                            scp -o StrictHostKeyChecking=no deploy/docker-compose.prod.yml "$DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_DIR/docker-compose.prod.yml"

                            ssh -o StrictHostKeyChecking=no "$DEPLOY_USER@$DEPLOY_HOST" "
                                set -e
                                cd '$DEPLOY_DIR'

                                if [ ! -f .env ]; then
                                    echo 'Missing $DEPLOY_DIR/.env on the server. Create it from deploy/.env.prod.example before deploying.' >&2
                                    exit 1
                                fi

                                cat > .images.env <<EOF
BACKEND_IMAGE=$BACKEND_IMAGE
FRONTEND_IMAGE=$FRONTEND_IMAGE
IMAGE_TAG=$IMAGE_TAG
EOF

                                echo '$REGISTRY_PASSWORD' | docker login '$REGISTRY' -u '$REGISTRY_USERNAME' --password-stdin
                                docker compose --env-file .env --env-file .images.env -f docker-compose.prod.yml pull
                                docker compose --env-file .env --env-file .images.env -f docker-compose.prod.yml up -d --remove-orphans
                                docker image prune -f
                            "
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout "$REGISTRY" || true'
        }
    }
}
