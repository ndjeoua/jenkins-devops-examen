def deploy(String ns, String port) {
    sh """
        helm upgrade --install movie-db charts -n ${ns} -f charts/values-movie-db.yaml --wait
        helm upgrade --install cast-db charts -n ${ns} -f charts/values-cast-db.yaml --wait
        helm upgrade --install cast-service charts -n ${ns} -f charts/values-cast.yaml --set image.tag=${TAG} --set service.nodePort=303${port} --wait
        helm upgrade --install movie-service charts -n ${ns} -f charts/values-movie.yaml --set image.tag=${TAG} --set service.nodePort=302${port} --wait
        curl -sf http://localhost:302${port}/api/v1/movies/docs > /dev/null
        curl -sf http://localhost:303${port}/api/v1/casts/docs > /dev/null
    """
}

pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'raoulndjeoua'
        TAG = "${env.GIT_COMMIT.take(7)}"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t $DOCKERHUB_USER/jenkins-exam-movie:$TAG movie-service'
                sh 'docker build -t $DOCKERHUB_USER/jenkins-exam-cast:$TAG cast-service'
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker compose -p jenkins-test up -d --build
                    for i in $(seq 1 30); do curl -sf http://localhost:8080/api/v1/casts/docs > /dev/null && break; sleep 2; done
                    curl -sf -X POST http://localhost:8080/api/v1/casts/ -H "Content-Type: application/json" -d '{"name": "Test", "nationality": "FR"}'
                    curl -sf http://localhost:8080/api/v1/movies/docs > /dev/null
                '''
            }
            post {
                always {
                    sh 'docker compose -p jenkins-test down -v'
                }
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $DOCKERHUB_USER/jenkins-exam-movie:$TAG
                        docker push $DOCKERHUB_USER/jenkins-exam-cast:$TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy dev') {
            steps { script { deploy('dev', '01') } }
        }

        stage('Deploy qa') {
            steps { script { deploy('qa', '02') } }
        }

        stage('Deploy staging') {
            steps { script { deploy('staging', '03') } }
        }

        stage('Deploy prod') {
            when { branch 'master' }
            steps {
                input message: 'Deployer en production ?', ok: 'Deployer'
                script { deploy('prod', '04') }
            }
        }
    }
}
