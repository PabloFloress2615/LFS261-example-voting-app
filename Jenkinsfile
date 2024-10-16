pipeline {
    agent none

    stages {
        stage('Install Dependencies for Vote') {
            when {
                changeset "**/vote/**"
            }
            agent {
                docker {
                    image 'python:3.11-slim'
                    args '--user root'
                }
            }
            steps {
                echo 'Installing dependencies for vote app...'
                dir('vote') {
                    sh "pip install -r requirements.txt"
                    sh "pip list" // Verificar que las dependencias estén instaladas
                }
            }
        }

        stage('Run Tests for Vote') {
            when {
                changeset "**/vote/**"
            }
            agent {
                docker {
                    image 'python:3.11-slim'
                    args '--user root'
                }
            }
            steps {
                echo 'Running unit tests on vote app...'
                dir('vote') {
                    sh 'pip install -r requirements.txt'
                    sh 'python -m nose2 -v'
                }
            }
        }
	stage('Vote Integration Test') {
	    agent any
	    when {
	        // Ejecuta este stage solo si hay cambios en la carpeta 'vote' y estamos en la rama 'master'
	        changeset "**/vote/**"
	        branch 'master'
	    }
	    steps {
	        echo 'Running Integration Tests on vote app'
	        dir('vote') {
	            // Ejecuta el script de integración llamado 'integration_test.sh'
	            sh 'sh integration_test.sh'
	        }
	    }
	}


        stage('Package Vote with Docker') {
            when {
                changeset "**/vote/**"
                branch 'master'
            }
            agent any
            steps {
                echo 'Packaging vote app with Docker...'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        def voteImage = docker.build("pabloflores2615/vote:v${env.BUILD_ID}", "./vote")
                        voteImage.push()
                        voteImage.push("${env.BRANCH_NAME}")
                        voteImage.push("latest")
                    }
                }
            }
        }

        stage('Install Dependencies for Result') {
            when {
                changeset "**/result/**"
            }
            agent {
                docker {
                    image 'node:18-slim'
                }
            }
            steps {
                echo 'Installing dependencies for result app...'
                dir('result') {
                    sh 'npm install'
                }
            }
        }

        stage('Test Result') {
            when {
                changeset "**/result/**"
            }
            agent {
                docker {
                    image 'node:18-slim'
                }
            }
            steps {
                echo 'Running Unit Tests on result app...'
                dir('result') {
                    sh 'npm test'
                }
            }
        }

        stage('Package Result with Docker') {
            when {
                changeset "**/result/**"
                branch 'master'
            }
            agent any
            steps {
                echo 'Packaging result app with Docker...'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        def resultImage = docker.build("pabloflores2615/result:v${env.BUILD_ID}", "./result")
                        resultImage.push()
                        resultImage.push("${env.BRANCH_NAME}")
                        resultImage.push("latest")
                    }
                }
            }
        }

        stage('Build Worker') {
            when {
                changeset "**/worker/**"
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Compiling worker app...'
                dir('worker') {
                    sh 'mvn compile'
                }
            }
        }

        stage('Test Worker') {
            when {
                changeset "**/worker/**"
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Running Unit Tests on worker app...'
                dir('worker') {
                    sh 'mvn clean test'
                }
            }
        }

        stage('Package Worker') {
            when {
                changeset "**/worker/**"
                branch 'master'
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Packaging worker app...'
                dir('worker') {
                    sh 'mvn package -DskipTests'
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage('Docker Package Worker') {
            when {
                changeset "**/worker/**"
                branch 'master'
            }
            agent any
            steps {
                echo 'Packaging worker app with Docker...'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        def workerImage = docker.build("pabloflores2615/worker:v${env.BUILD_ID}", "./worker")
                        workerImage.push()
                        workerImage.push("${env.BRANCH_NAME}")
                        workerImage.push("latest")
                    }
                }
            }
        }

        stage('Sonarqube') {
            agent any
            when {
                branch 'master'
            }
            environment {
                sonarpath = tool 'sonar-instavote'
            }
            steps {
                echo 'Running Sonarqube Analysis...'
                withSonarQubeEnv('sonar-instavote') {
                    sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                branch 'master'
            }
            agent any
            steps {
                echo 'Deploying Instavote stack to development...'
                sh 'docker compose up -d'
            }
        }
	stage('Trigger deployment') {
	    agent any
	    steps {
	        script {
	            def commit = env.GIT_COMMIT // Asignar el valor de GIT_COMMIT a una variable local
	            echo "Current GIT_COMMIT: ${commit}"
	            echo "Triggering deployment"
	            
	            // Aquí se llama a otro job de Jenkins para hacer el despliegue
	            build job: 'deployment', 
	                  parameters: [string(name: 'COMMIT', value: commit)]
	        }
	    }
	}

    }

    post {
        always {
            echo 'Monopipe execution is complete for vote, result, and worker apps.'
        }
    }
}
