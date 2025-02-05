pipeline {
    agent any

    stages {
        
        stage('Checkout Code') {
            steps {
                script {
                    checkout scmGit(
                        branches: [[name: '**']],  // Fetches all branches
                        userRemoteConfigs: [[url: 'https://github.com/tsaekao/nodegoat.git']]
                    )
                }
            }
        }

        stage('Package Application') {
            steps {
                sh 'curl -fsS https://tools.veracode.com/veracode-cli/install | sh'
                sh './veracode package -s . -o veracode-artifact -a trust'
            }
        }

        stage('Upload SAST') {
            when {
                expression { env.CHANGE_TARGET == 'master' }  // Runs only when there's a pull request to master
            }
            steps {
                withCredentials([usernamePassword(credentialsId: '3dd8eb7f-547e-4b79-8a6c-f39954a06c63', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    veracode applicationName: 'nodegoat',
                    criticality: 'VeryHigh',
                    scanName: '$buildnumber',
                    uploadIncludesPattern: 'veracode-artifact/',
                    vid: USER,
                    vkey: PASS,
                    deleteIncompleteScanLevel: '2'
                    sh "pwd"
                    echo "SAST Scan Triggered"
                }
            }
        }

        stage('SCA Agent-Based Scan') {
            steps {
                withCredentials([string(credentialsId: 'SRCCLR_API_TOKEN', variable: 'SRCCLR_API_TOKEN')]) {
                    catchError(buildResult: 'SUCCESS', message: 'SUCCESS') {
                        sh "srcclr scan . --allow-dirty" 
                        echo "SCA Agent-Based Scan Completed"
                    }
                }
            }
        }

        stage('Link SCA Project to Application Profile') {
            steps {
                sh '''
                    apt-get update && apt-get install -y python3 python3-pip || true
                    pip3 install veracode-api-signing --upgrade
                '''
                withCredentials([usernamePassword(credentialsId: '3dd8eb7f-547e-4b79-8a6c-f39954a06c63', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh '''
                        python3 link_sca_project.py --workspace_name "Jenkins Demo" --project_name "tsaekao/nodegoat" --application_profile "nodegoat" --api_id "${USER}" --api_key "${PASS}"
                    '''
                }
            }
        }
        
        stage('Veracode Pipeline Scan') {
            when {
                expression { env.BRANCH_NAME != 'master' }  // Runs when there’s a commit on any non-master branch
            }
            steps {
                withCredentials([usernamePassword(credentialsId: '3dd8eb7f-547e-4b79-8a6c-f39954a06c63', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh '''
                        export VERACODE_API_KEY_ID=${USER}
                        export VERACODE_API_KEY_SECRET=${PASS}
                        ./veracode static scan veracode-artifact/*-js.zip --fail-on-severity 'Very High, High' || true
                        git clone https://github.com/tjarrettveracode/veracode-pipeline-mitigation
                        pip install -r veracode-pipeline-mitigation/requirements.txt
                        python veracode-pipeline-mitigation/vcpipemit.py -an nodegoat --results 'results.json'
                        mv baseline-*.json baseline.json
                        ./veracode static scan veracode-artifact/*-js.zip --fail-on-severity 'Very High, High' --baseline-file baseline.json
                    '''
                }
            }
        }
    }

    post { 
        always { 
            cleanWs()
        }
    }
}
