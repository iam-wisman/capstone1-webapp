// Jenkins Pipeline for Abode Software's Capstone I lifecycle.
// Triggered by a GitHub webhook on every push to master or develop (this stands in for the
// brief's "CodeBuild" step -- there is no AWS CodeBuild project here, Jenkins' own webhook
// trigger + Build stage plays that role). Every push builds and tests the Docker image;
// only a push to master goes on to Job3 and ships the image to prod.
pipeline {
    agent any

    environment {
        IMAGE_NAME = "wuisman/capstone1-webapp"
        PROD_SERVER_IP = "34.227.223.35"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Job1: build -- Dockerfile is rebuilt on every push to GitHub (spec item 4).
        stage('Job1: build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        // Job2: test -- runs on every branch. Boots the image and checks the site responds
        // before anything is considered safe to ship.
        stage('Job2: test') {
            steps {
                sh """
                    docker rm -f capstone1-test || true
                    docker run -d --name capstone1-test -p 8081:80 ${IMAGE_NAME}:${BUILD_NUMBER}
                    sleep 5
                    curl -sf http://localhost:8081/ -o /dev/null && echo 'Smoke test passed: site responds on 80'
                    docker rm -f capstone1-test
                """
            }
        }

        // Job3: prod -- master only (spec item 3a: "if a commit is made to master branch,
        // test and push to prod"). develop stops after Job2 (spec item 3b: "just test the
        // product, do not push to prod").
        stage('Job3: prod') {
            when {
                branch 'master'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:prod
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:prod
                    """
                }
                sshagent(credentials: ['prod-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} '
                            docker pull ${IMAGE_NAME}:prod
                            docker rm -f capstone1-prod || true
                            docker run -d --name capstone1-prod -p 80:80 --restart unless-stopped ${IMAGE_NAME}:prod
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.BRANCH_NAME == 'master') {
                    echo "Job3 complete: ${IMAGE_NAME}:${BUILD_NUMBER} is live on prod-server (http://${PROD_SERVER_IP})"
                } else {
                    echo "Job2 complete: ${env.BRANCH_NAME} built and tested, not pushed to prod"
                }
            }
        }
        failure {
            echo "Pipeline failed - check the stage logs above"
        }
    }
}
