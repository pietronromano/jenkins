# Pipeline Script pasted directly in Jenkins Pipeline UI

# Script
pipeline {
    # agent any
    agent {label 'DockerAgent'}

    stages {
        stage('Hello') {

            steps {
                sh '''
                    file_name=time_log.txt
                    current_time=$(date '+%Y.%m.%d-%H.%M.%S')
                    echo $current_time
                    echo 'Writing to $current_time to $file_name'
                    echo $current_time >> $file_name
                    echo 'Getting User:'
                    whoami
                    echo 'PATH:'
                    echo $PATH
                '''
            }
        }
    }
}
