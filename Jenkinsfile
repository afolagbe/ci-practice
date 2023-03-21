def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]
pipeline {
    agent any
    tools {
        maven 'MAVEN3'
        jdk 'OracleJDK8'
    }
    environment {
        SNAP_REPO = 'vpro-snapshots'
        NEXUS_USER = 'admin'
        NEXUS_PASS = credentials('nexuspass')
        DOCKER_PASS= credentials('dockerpass')
        RELEASE_REPO = 'vpro-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_IP = '172.31.8.109'
        NEXUS_PORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONARSERVER = 'SONARSERVER'
        SONARSCANNER = 'sonar-scanner'
    }
    stages{
        stage('BUILD'){
            steps{
                sh 'mvn -s settings.xml install -DskipTests'
            }
            post {
                success {
                    echo "Now Archiving"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage ('TEST'){
            steps {
                sh 'mvn -s settings.xml test'
            }
        }
        stage ('CHECKSTYLE ANALYSIS'){
            steps{
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }
        stage ('SONAR ANALYSIS') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
            }
        }
        stage ('QUALITY GATE') {
            steps {
                timeout (time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage ('UPLOAD ARTIFACT') {
            steps {
                    nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                    [artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war']
                     ]
                    )
                     }
        }
        stage('ANSIBLE DEPLOY to STAGING'){
            steps{
                ansiblePlaybook([
                    inventory: 'ansible/stage.inventory',
                    playbook: 'ansible/site.yml',
                    installation: 'ansible',
                    colorized: true,
                    credentialsId: 'dockerlogin',
                    disableHostKeyChecking: true,
                    extraVars:[
                        USER: 'admin',
                        PASS: "${NEXUS_PASS}",
                        nexusip: "${NEXUS_IP}",
                        reponame: "${RELEASE_REPO}",
                        groupid: 'QA',
                        artifactid: 'vproapp',
                        build: "${env.BUILD_ID}",
                        time: "${env.BUILD_TIMESTAMP}",
                        vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                        dockerPass: "${DOCKER_PASS}"

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
