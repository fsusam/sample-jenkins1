def bob = 'docker run --rm --env APP_PATH=$PWD -v $PWD:$PWD -v /var/run/docker.sock:/var/run/docker.sock armdocker.rnd.ericsson.se/proj-eson/bob:0.0.5 --verbose'
def imagePushed = false

pipeline {
    agent {
        node {
            label SLAVE
        }
    }
    tools {
        jdk 'jdk8'
        maven "Maven 3.5.4"
    }
    options {
        skipDefaultCheckout true
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }
    parameters {
        string(name: 'SETTINGS_CONFIG_FILE_NAME', defaultValue: 'eson_settings.xml')
        string(name: 'KUBERNETES_CONFIG_FILE_NAME', defaultValue: 'kube.config.zenith')
        string(name: 'HOSTS_FILE_NAME', defaultValue: 'hosts.zenith')
    }
    environment {
        HELM_CMD = "docker run --rm -v ${env.WORKSPACE}/resources/hosts:/etc/hosts -v ${env.WORKSPACE}/.kube/config:/root/.kube/config -v ${env.WORKSPACE}/helm-home:/root/.helm -v ${env.WORKSPACE}:${env.WORKSPACE} linkyard/docker-helm:2.8.2"
        SERVICE_NAME = "cm-data-loading-service-enm"
        CHART_REPO_URL = "https://arm.epk.ericsson.se/artifactory/proj-eson-${SERVICE_NAME}-helm/"
        UPGRADE_RELEASE_NAME = "eric-${SERVICE_NAME}-upgrade"
        INSTALL_RELEASE_NAME = "eric-${SERVICE_NAME}-install"
        UPGRADE_NAMESPACE = "${UPGRADE_RELEASE_NAME}"
        INSTALL_NAMESPACE = "${INSTALL_RELEASE_NAME}"
        CHART_DIRECTORY = "${WORKSPACE}/helm-target/$SERVICE_NAME/*.tgz"
    }
    stages {
        stage('Clean up Workspace From Previous Build') {
            steps {
                script {
                    sh 'sudo chmod -fR 777 "${WORKSPACE}" &&\
                        sudo rm -Rf ./*'
                }
            }
        }
        stage('Checkout From scm') {
            steps {
                checkout scm
            }
        }
        stage('Inject Maven Settings.xml') {
            steps {
                sh 'echo Injecting Maven Settings.xml'
                configFileProvider([configFile(fileId: "${env.SETTINGS_CONFIG_FILE_NAME}", targetLocation: "${HOME}/.m2/settings.xml")]) {
                }
            }
        }
        stage('Inject ECCD Configuration Files') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                sh 'echo Injecting Kubernetes config file'
                configFileProvider([configFile(fileId: "${env.KUBERNETES_CONFIG_FILE_NAME}", targetLocation: "${env.WORKSPACE}/.kube/")]) {
                }
                sh 'echo Injecting Hosts file to identify ECCD cluster'
                configFileProvider([configFile(fileId: "${env.HOSTS_FILE_NAME}", targetLocation: "${env.WORKSPACE}/resources/")]) {
                }
            }
        }
        stage('Ensure no Previous Releases Exist on ECCD') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                sh 'mkdir helm-home'
                sh '${HELM_CMD} delete --purge $INSTALL_RELEASE_NAME || true'
                sh '${HELM_CMD} delete --purge $UPGRADE_RELEASE_NAME || true'
            }
        }
        stage('Pre Code Review') {
            when {
                expression { env.PRE_CODE_REVIEW }
            }
            steps {
                // CI Common post step script
                sh "/proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/pre_step/preCodeReview_pre_step.sh"
                sh 'mvn -V jacoco:prepare-agent install jacoco:report pmd:pmd'
                // CI Common post step script
                sh "/proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/post_step/preCodeReview_post_step.sh"
            }
        }
        stage('Acceptance Test') {
            when {
                expression { env.ACCEPTANCE || env.SERVICE_RELEASE }
            }
            steps {
                sh 'echo "CI Common pre step scripts" &&\
                    /proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/pre_step/acceptance_pre_step.sh &&\
                    ls /proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/pre_step/ &&\
                    echo "Download CI scripts" &&\
                    curl -o ci-job-scripts.zip "https://arm101-eiffel004.lmera.ericsson.se:8443/nexus/service/local/repositories/releases/content/com/ericsson/oss/services/test/ci-job-scripts/1.0.159/ci-job-scripts-1.0.159.zip" &&\
                    unzip ci-job-scripts.zip &&\
                    echo "Bring up containers" &&\
                    cd testsuite/integration/jee &&\
                    docker-compose up --build -d &&\
                    $WORKSPACE/ci-job-scripts/wait-jboss.sh cm-data-loading-service-enm_wildfly # Waits until the Wildfly instance (with the supplied name) is ready'
                script {
                    if (env.SERVICE_RELEASE) {
                        sh 'mvn clean install -V -Dts'
                    } else {
                        sh 'echo ${BOM_TEST_ARTIFACT_VERSION}'
                        sh 'mvn clean install -V -Dts -Dversion.son-test-bom=${BOM_TEST_ARTIFACT_VERSION}'
                    }
                }
            }
        }
        stage('Release Artifacts to Nexus') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                // Get Release Version to pass to BOM
                script {
                    pom = readMavenPom file: 'pom.xml'
                    RELEASE_VERSION = pom.version.substring(0, pom.version.indexOf('-SNAPSHOT'))
                }
                // CI Common pre step script
                sh "/proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/pre_step/preCodeReview_pre_step.sh"
                sh "mvn -V -Dresume=false release:prepare release:perform  -DpreparationGoals='install' -Dgoals='clean deploy pmd:pmd jacoco:report' -DlocalCheckout=true"
                // Modify the build description
                script {
                    try {
                        def matcher = manager.getLogMatcher('.*\\[INFO\\] \\[INFO\\] Building \\[cm-data-loading-service-enm\\] (JEE6 project) (.*)\\s+\\[\\d+\\/\\d+\\]')
                        if (matcher != null) {
                            currentBuild.description = matcher.group(2)
                        }
                    }
                    catch (ignored) {
                    }
                }
                // CI Common post step script
                sh "/proj/ciexadm200/tools/utils/scripts/common_pre_post_jenkins_scripts/post_step/preCodeReview_post_step.sh"
            }
        }
        stage('Checkout Release') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                script {
                    sh "git checkout $SERVICE_NAME-$RELEASE_VERSION"
                }
            }
        }
        stage('Lint Chart') {
            when {
                expression { env.PRE_CODE_REVIEW || env.SERVICE_RELEASE }
            }
            steps {
                sh "${bob} lint"
            }
        }
        stage('Build Image and Chart') {
            when {
                expression { env.PRE_CODE_REVIEW || env.SERVICE_RELEASE }
            }
            steps {
                sh "${bob} image --env RELEASE=true"
            }
        }
        stage('Push Image For Chart Dependency') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                script {
                    //The following command retrieves the semantic bob version from the name of the created helm chart.
                    sh 'TAG="$(basename $(ls -d $WORKSPACE/helm-target/$SERVICE_NAME/*) .tgz | rev | cut -d- -f-2 | rev )" && \
                        docker push armdocker.rnd.ericsson.se/proj-eson/$SERVICE_NAME:$TAG'
                    imagePushed = true
                }
            }
        }
        stage('Prepare Helm') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                sh '${HELM_CMD} init --client-only'
                sh '${HELM_CMD} repo add ${SERVICE_NAME}-repo $CHART_REPO_URL'
                sh '${HELM_CMD} repo update'
            }
        }
        stage('Upgrade and Install Chart') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                parallel(
                        "Upgrade Chart": {
                            // Install known good baseline
                            sh '${HELM_CMD} upgrade --install ${UPGRADE_RELEASE_NAME} ${SERVICE_NAME}-repo/${SERVICE_NAME} --set install.database=true --set eric-oss-repository-data.postgresPassword=ABC123  --namespace ${UPGRADE_NAMESPACE} --devel --wait --timeout 600'

                            // Upgrade to newly created chart
                            sh '${HELM_CMD} upgrade --install ${UPGRADE_RELEASE_NAME} ${CHART_DIRECTORY} --set install.database=true --set eric-oss-repository-data.postgresPassword=ABC123 --namespace ${UPGRADE_NAMESPACE} --devel --wait --timeout 600'
                        },
                        "Install Chart": {
                            sh '${HELM_CMD} install --name ${INSTALL_RELEASE_NAME} ${CHART_DIRECTORY} --set install.database=true --set eric-oss-repository-data.postgresPassword=ABC123 --namespace ${INSTALL_NAMESPACE} --wait --timeout 600'
                        }
                )
            }
        }
        stage('Remove the Service Installation') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                sh '${HELM_CMD} delete --purge $INSTALL_RELEASE_NAME'
                sh '${HELM_CMD} delete --purge $UPGRADE_RELEASE_NAME'
            }
        }
        stage('Push Image and Chart') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                script {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'ejenksonomArtifactoryApiKey', usernameVariable: 'USERNAME', passwordVariable: 'APIKEY']]) {
                        sh "${bob} push --env RELEASE=true"
                        sh "docker login -u ${USERNAME} -p ${APIKEY} https://armdocker.rnd.ericsson.se/  && \
                            docker push armdocker.rnd.ericsson.se/proj-eson/${SERVICE_NAME}:latest"
                    }
                }
            }
        }
        stage('Generate Input for eSON Staging') {
            when {
                expression { env.SERVICE_RELEASE }
            }
            steps {
                sh "${bob} generate-output-parameters --env RELEASE=true"
                archiveArtifacts 'artifact.properties'
            }
        }
    }
    post {
        always {
            script {
                if (env.ACCEPTANCE || env.SERVICE_RELEASE) {
                    sh 'docker ps -a &&\
                        echo "Copy server.log to host" &&\
                        wildflyContainerId=$(docker ps -a | grep "wildfly" | cut -d " " -f 1 | xargs) &&\
                        docker cp ${wildflyContainerId}:/ericsson/3pp/wildfly/standalone/log/server.log . &&\
                        echo "#############################################" &&\
                        cd testsuite/integration/jee &&\
                        docker-compose kill || echo &&\
                        docker-compose down -v || echo &&\
                        docker ps -qa | xargs docker rm -fv >& /dev/null || echo &&\
                        docker volume prune'
                    archiveArtifacts '*.log'
                } else {
                    step([$class: 'JacocoPublisher', execPattern: '**/**.exec'])
                    step([$class: 'PmdPublisher', pattern: '**/target/pmd.xml', failedTotalAll: '0'])
                }
            }
        }
        success {
            // Update BOM
            script {
                if (env.SERVICE_RELEASE) {
                    sh 'echo ${SERVICE_NAME}'
                    build job: 'son-test-bom_Update', propogate: false, wait: false, parameters: [[$class: 'StringParameterValue', name: 'SERVICE_NAME', value: "${SERVICE_NAME}"], [$class: 'StringParameterValue', name: 'SERVICE_VERSION', value: RELEASE_VERSION]]
                }
            }
        }
        failure {
            step([$class: 'ClaimPublisher'])
            script {
                if (!env.PRE_CODE_REVIEW) {
                    emailext to: 'PDLESONTEA@pdl.internal.ericsson.com', recipientProviders: [culprits(), developers(), requestor(), brokenBuildSuspects()], subject: "FAILURE: ${currentBuild.fullDisplayName}", body: "<b>Jenkins job failed:</b><br><br>Project: ${env.JOB_NAME}<br>Build Number: ${env.BUILD_NUMBER}<br>${env.BUILD_URL}", mimeType: 'text/html'
                }
            }
            // Remove image from Artifactory
            script {
                //The following command retrieves the semantic bob version from the name of the created helm chart.
                if (imagePushed) {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'ejenksonomArtifactoryApiKey', usernameVariable: 'USERNAME', passwordVariable: 'APIKEY']]) {
                        sh 'TAG="$(basename $(ls -d $WORKSPACE/helm-target/$SERVICE_NAME/*) .tgz | rev | cut -d- -f-2 | rev )" && \
                            curl -u ${USERNAME}:${APIKEY} -X DELETE https://arm.epk.ericsson.se/artifactory/docker-v2-global-local/proj-eson/$SERVICE_NAME/$TAG'
                    }
                }
            }
        }
    }
}
