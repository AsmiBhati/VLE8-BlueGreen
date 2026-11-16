pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        IMAGE = 'asmibhati/vle8-bluegreen:green-v2'
        EC2_HOST = '54.208.213.104'
        REMOTE_DIR = '/home/ubuntu/kubernetes'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Green Image') {
            steps {
                sh '''
                    docker build \
                      --build-arg PAGE=green.html \
                      -t $IMAGE ./app
                '''
            }
        }

        stage('Push Green Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Green') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ec2-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        chmod 600 "$SSH_KEY"

                        scp -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          kubernetes/green.yaml \
                          "$SSH_USER@$EC2_HOST:$REMOTE_DIR/green.yaml"

                        ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          "$SSH_USER@$EC2_HOST" \
                          "sudo k3s kubectl apply -f $REMOTE_DIR/green.yaml && sudo k3s kubectl rollout status deployment/myapp-green --timeout=180s"
                    '''
                }
            }
        }

        stage('Approve Traffic Switch') {
            steps {
                input message: 'Green deployment is ready. Switch application traffic from Blue to Green?',
                      ok: 'Switch to Green'
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ec2-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          "$SSH_USER@$EC2_HOST" \
                          "sudo k3s kubectl patch service myapp-service -p '{\\"spec\\":{\\"selector\\":{\\"app\\":\\"myapp\\",\\"color\\":\\"green\\"}}}' && sudo k3s kubectl get service myapp-service -o yaml"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Green deployment completed and traffic switched successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the failed stage. Blue remains the active version unless the traffic switch already completed.'
        }
        aborted {
            echo 'Pipeline aborted. If approval was declined or timed out, Blue remains active.'
        }
    }
}
