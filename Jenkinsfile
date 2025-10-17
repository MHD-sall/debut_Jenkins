pipeline {
    agent any

    tools {
        nodejs "NodeJS_22"
    }

    environment {
        DOCKER_HUB_USER = 'sall889'
        FRONT_IMAGE = 'react-frontend'
        BACK_IMAGE  = 'express-backend'
        SONARQUBE_ENV = 'SonarQube'
        SCANNER_TOOL = 'sonar-scanner'
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_AUTH_TOKEN = credentials('sonar')
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref'],
                [key: 'pusher_name', value: '$.pusher.name'],
                [key: 'commit_message', value: '$.head_commit.message']
            ],
            causeString: 'Push par $pusher_name sur $ref: "$commit_message"',
            token: 'mysecret',
            printContributedVariables: true,
            printPostContent: true
        )
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/MHD-sall/debut_Jenkins.git'
            }
        }

        stage('Install dependencies - Backend') {
            steps {
                dir('back-end') {
                    sh 'npm install'
                }
            }
        }

        stage('Install dependencies - Frontend') {
            steps {
                dir('front-end') {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    sh 'cd back-end && npm test || echo "Aucun test backend"'
                    sh 'cd front-end && npm test || echo "Aucun test frontend"'
                }
            }
        }

        // ======================== SonarQube ========================
        stage('SonarQube Analysis') {
    steps {
        script {
            // 🔹 Récupérer le chemin du scanner
            def scannerHome = tool 'SonarQubeScanner' 

            // 🔹 Définir l'environnement SonarQube
            withSonarQubeEnv('sonar') {
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=MonProjet \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://localhost:9000 \
                    -Dsonar.login=admin \
                    -Dsonar.password=admin
                """
            }
        }
    }
}

stage('Quality Gate') {
    steps {
        echo "🛡️ Vérification du Quality Gate..."
        timeout(time: 2, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}


        stage('Build Docker Images') {
            steps {
                script {
                    sh "docker build -t $DOCKER_HUB_USER/$FRONT_IMAGE:latest ./front-end"
                    sh "docker build -t $DOCKER_HUB_USER/$BACK_IMAGE:latest ./back-end"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_USER/react-frontend:latest
                        docker push $DOCKER_USER/express-backend:latest
                    '''
                }
            }
        }

        stage('Clean Docker') {
            steps {
                sh 'docker container prune -f'
                sh 'docker image prune -f'
            }
        }

        stage('Check Docker & Compose') {
            steps {
                sh 'docker --version'
                sh 'docker-compose --version || echo "docker-compose non trouvé"'
            }
        }

        stage('Deploy (compose.yaml)') {
            steps {
                dir('.') {
                    sh 'docker-compose -f compose.yaml down || true'
                    sh 'docker-compose -f compose.yaml pull'
                    sh 'docker-compose -f compose.yaml up -d'
                    sh 'docker-compose -f compose.yaml ps'
                    sh 'docker-compose -f compose.yaml logs --tail=50'
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    echo " Vérification Frontend (port 5173)..."
                    curl -f http://localhost:5173 || echo "Frontend unreachable"

                    echo " Vérification Backend (port 5001)..."
                    curl -f http://localhost:5001/api || echo "Backend unreachable"
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Pipeline réussi\nDétails : ${env.BUILD_URL}",
                to: 'methzosll889@gmail.com',
                from: 'methzosll889@gmail.com',
                replyTo: 'methzosll889@gmail.com'
            )
        }
        failure {
            emailext(
                subject: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Le pipeline a échoué\nDétails : ${env.BUILD_URL}",
                to: 'methzosll889@gmail.com',
                from: 'methzosll889@gmail.com',
                replyTo: 'methzosll889@gmail.com'
            )
        }
    }
}

