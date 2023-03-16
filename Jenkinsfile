def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]
pipeline {
    agent any
    environment {
        NEXUS_USER = 'admin'
        NEXUS_PASS = credentials('nexuspass')
        RELEASE_REPO = 'vpro-release'
        NEXUS_IP = '172.31.23.174'
    }
    stages{
        stage('SETUP PARAMETERS'){
            steps{
                script{
                    properties([
                        string(
                            defaultValue: '',
                            name: 'BUILD'
                        ),
                        string(
                            defaultValue: '',
                            name: "TIME"
                        )
                    ])
                }
            }
        }
        
        stage('ANSIBLE DEPLOY to PRODUCTION'){
            steps{
                ansiblePlaybook([
                    inventory: 'ansible/prod.inventory',
                    playbook: 'ansible/site.yml',
                    installation: 'ansible',
                    colorized: true,
                    credentialsId: 'applogin-prod',
                    disableHostKeyChecking: true,
                    extraVars:[
                        USER: 'admin',
                        PASS: "${NEXUS_PASS}",
                        nexusip: "${NEXUS_IP}",
                        reponame: "${RELEASE_REPO}",
                        groupid: 'QA',
                        artifactid: 'vproapp',
                        build: "${env.BUILD}",
                        time: "${env.TIME}",
                        vprofile_version: "vproapp-${env.BUILD}-${env.TIME}.war"

                    ]
                ])
            }
        }
    }
    post {
        always {
            echo 'slack notifications.'
            slackSend channel: '#project-ci',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
