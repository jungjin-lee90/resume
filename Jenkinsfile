pipeline {
    agent any
	
    options {
        disableConcurrentBuilds()  // 동시 실행 방지
	ansiColor('xterm')
        timestamps()
    }

    parameters {
        string(name: 'PORT', defaultValue: '8502', description: 'Host port to expose')
        string(name: 'IMAGE_NAME', defaultValue: 'resume', description: 'Docker image and container name')
    }

    stages {
	stage('Set Version Tag') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: "date +%Y%m%d-%H%M%S",
                        returnStdout: true
                    ).trim()
                }
            }
        }
	
	stage('Checkout') {
            steps {
                checkout scm
            }
        }

	stage('Code Diff with Color') {
    	    steps {
        	ansiColor('xterm') {
            	    sh 'git diff --color=always HEAD~1'
            	}
    	    }
	}	
	
	stage('Label Build') {
    	    steps {
        	script {
            	    def commitMsg = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()
            	    def commitAuthor = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
            	    currentBuild.displayName = "#${env.BUILD_NUMBER} | ${commitMsg} by ${commitAuthor}"
        	}
    	    }
	}	
	
	stage('Recent Commits') {
    	    steps {
        	echo "최근 커밋 내역:"
        	sh "git log -3 --pretty=format:'%h | %s by %an' --abbrev-commit"
    	    }
	}	
	
	stage('Build Docker Image') {
            steps {
                sh """
                echo "Building Docker image with tag: ${env.IMAGE_TAG}"
                docker build -t ${params.IMAGE_NAME}:${env.IMAGE_TAG} .
                docker tag ${params.IMAGE_NAME}:${env.IMAGE_TAG} ${params.IMAGE_NAME}:latest
                """
            }
        }
	
	stage('Clean old Docker images') {
    	    steps {
        	script {
            	    def imageName = params.IMAGE_NAME
            	    def currentImageId = sh(
                    	script: "docker inspect --format='{{.Image}}' \$(docker ps -q --filter name=${imageName}) || true",
                    	returnStdout: true
            	    ).trim()

            	    sh """
            	    echo "[*] Cleaning up unused Docker images (except running one)..."
            	    docker images ${imageName} --format '{{.ID}}' | grep -v "${currentImageId}" | tail -n +4 | xargs -r docker rmi || true
            	    """
                }
    	    }
	}

	stage('Release Port if Used') {
   	     steps {
        	script {
            	    def port = params.PORT
            	    sh """
                	echo "[*] Checking if port ${port} is in use..."
                	CONFLICT=\$(docker ps -q --filter "publish=${port}")
                	if [ ! -z "\$CONFLICT" ]; then
                    	    echo "[!] Port ${port} is in use. Removing conflicting container(s)..."
                    	    docker rm -f \$CONFLICT
                	else
                    	    echo "[+] Port ${port} is free."
                	fi
            	    """
        	}
    	    }
	}
	stage('Run Container') {
    	    steps {
        	sh """
        	echo "[*] Stopping and removing existing container (if any)..."
        	docker rm -f ${params.IMAGE_NAME} || true

        	echo "[*] Running container with healthcheck..."
        	docker run -d \
          	--name ${params.IMAGE_NAME} \
          	--restart always \
		--health-cmd="curl -f http://localhost || exit 1"
          	--health-interval=30s \
          	--health-timeout=5s \
          	--health-retries=3 \
          	-p ${params.PORT}:80 \
          	${params.IMAGE_NAME}:${env.IMAGE_TAG}
        	"""
    	    }
	}
	
	stage('GitHub Release') {
    	    when {
        	expression { return env.IMAGE_TAG != null }
    	    }
    	    steps {
        	withCredentials([string(credentialsId: 'github-pat', variable: 'GH_TOKEN')]) {
            	    withEnv(["IMAGE_TAG=${env.IMAGE_TAG}"]) {
                	sh '''#!/bin/bash
                    	    echo "$GH_TOKEN" | gh auth login --with-token
                    	    gh release create "v$IMAGE_TAG" \
                      	    --title "Release v$IMAGE_TAG" \
                      	    --notes "자동 배포된 릴리즈입니다." \
                      	    --repo jungjin-lee90/LJM
                	'''
            	    }
        	}
    	    }
	}
    }

    post {
        success {
            echo "Build and deployment completed: ${params.IMAGE_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            echo "Build failed."
        }
    }
}
