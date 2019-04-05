/******************************************************************************
 * NOTICE                                                                     *
 *                                                                            *
 * This software (or technical data) was produced for the U.S. Government     *
 * under contract, and is subject to the Rights in Data-General Clause        *
 * 52.227-14, Alt. IV (DEC 2007).                                             *
 *                                                                            *
 * Copyright 2019 The MITRE Corporation. All Rights Reserved.                 *
 ******************************************************************************/

/******************************************************************************
 * Copyright 2019 The MITRE Corporation                                       *
 *                                                                            *
 * Licensed under the Apache License, Version 2.0 (the "License");            *
 * you may not use this file except in compliance with the License.           *
 * You may obtain a copy of the License at                                    *
 *                                                                            *
 *    http://www.apache.org/licenses/LICENSE-2.0                              *
 *                                                                            *
 * Unless required by applicable law or agreed to in writing, software        *
 * distributed under the License is distributed on an "AS IS" BASIS,          *
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.   *
 * See the License for the specific language governing permissions and        *
 * limitations under the License.                                             *
 ******************************************************************************/

// Jenkins Global Variables Reference: https://opensource.triology.de/jenkins/pipeline-syntax/globals

// Get build parameters.
def imageTag = env.getProperty("image_tag")
def emailRecipients = env.getProperty("email_recipients")

def openmpfDockerBranch = env.getProperty("openmpf_docker_branch")
def openmpfProjectsBranch = env.getProperty("openmpf_projects_branch")
def openmpfBranch = env.getProperty("openmpf_branch")
def openmpfComponentsBranch = env.getProperty("openmpf_components_branch")
def openmpfContribComponentsBranch = env.getProperty("openmpf_contrib_components_branch")
def openmpfCppComponentSdkBranch = env.getProperty("openmpf_cpp_component_sdk_branch")
def openmpfJavaComponentSdkBranch = env.getProperty("openmpf_java_component_sdk_branch")
def openmpfPythonComponentSdkBranch = env.getProperty("openmpf_python_component_sdk_branch")
def openmpfBuildToolsBranch = env.getProperty("openmpf_build_tools_branch")

def buildPackageJson = env.getProperty("build_package_json")
def buildOpenmpf = env.getProperty("build_openmpf").toBoolean()
def runGTests = env.getProperty("run_gtests").toBoolean()
def runMvnTests = env.getProperty("run_mvn_tests").toBoolean()
def mvnTestOptions = env.getProperty("mvn_test_options")
def buildRuntimeImages = env.getProperty("build_runtime_images").toBoolean()
def pushRuntimeImages = env.getProperty("push_runtime_images").toBoolean()

def dockerRegistryHost = env.getProperty("docker_registry_host")
def dockerRegistryPort = env.getProperty("docker_registry_port")
def dockerRegistryCredId = env.getProperty("docker_registry_cred_id")
def jenkinsNodes = env.getProperty("jenkins_nodes")
def extraTestDataPath = env.getProperty("extra_test_data_path")
// def buildNum = env.getProperty("BUILD_NUMBER")
// def workspacePath = env.getProperty("WORKSPACE")

// These properties are for building with custom components
def buildCustomComponents = env.getProperty("build_custom_components").toBoolean()
def openmpfCustomRepoCredId = env.getProperty('openmpf_custom_repo_cred_id')
def openmpfCustomDockerRepo = env.getProperty("openmpf_custom_docker_repo")
def openmpfCustomDockerBranch = env.getProperty("openmpf_custom_docker_branch")
def openmpfCustomComponentsRepo = env.getProperty("openmpf_custom_components_repo")
def openmpfCustomComponentsSlug = env.getProperty("openmpf_custom_components_slug")
def openmpfCustomComponentsBranch = env.getProperty("openmpf_custom_components_branch")
def openmpfCustomSystemTestsRepo = env.getProperty("openmpf_custom_system_tests_repo")
def openmpfCustomSystemTestsSlug = env.getProperty("openmpf_custom_system_tests_slug")
def openmpfCustomSystemTestsBranch = env.getProperty("openmpf_custom_system_tests_branch")

// These properties are for applying custom configurations to images
def applyCustomConfig = env.getProperty("apply_custom_config").toBoolean()
def openmpfConfigRepoCredId = env.getProperty('openmpf_config_repo_cred_id')
def openmpfConfigDockerRepo = env.getProperty("openmpf_config_docker_repo")
def openmpfConfigDockerBranch = env.getProperty("openmpf_config_docker_branch")

// These properties are for posting the Jenkins build status to GitHub
def postOpenmpfDockerBuildStatus = env.getProperty("post_openmpf_docker_build_status").toBoolean()
def githubAuthToken = env.getProperty("github_auth_token")

// SHAs
def openmpfDockerSha
def openmpfSha
def openmpfComponentsSha
def openmpfContribComponentsSha
def openmpfCppComponentSdkSha
def openmpfJavaComponentSdkSha
def openmpfPythonComponentSdkSha
def openmpfBuildToolsSha

// Labels
def buildDate
def buildShas
def openmpfCustomDockerSha
def openmpfCustomComponentsSha
def openmpfCustomSystemTestsSha
def openmpfConfigDockerSha

node(jenkinsNodes) {
wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) { // show color in Jenkins console
    def buildException

    // Rename the named volumes and networks to be unique to this Jenkins build pipeline
    def buildSharedDataVolumeSuffix = 'shared_data_' + currentBuild.projectName
    def buildSharedDataVolume = 'openmpf_' + buildSharedDataVolumeSuffix

    def buildMySqlDataVolumeSuffix = 'mysql_data_' + currentBuild.projectName
    def buildMySqlDataVolume = 'openmpf_' + buildMySqlDataVolumeSuffix

    def buildNetworkSuffix = 'compose_overlay_' + currentBuild.projectName
    def buildNetwork = 'openmpf_' + buildNetworkSuffix

    try {
        buildDate = getTimestamp()

        // Clean up last run
        sh 'docker volume rm -f ' + buildSharedDataVolume + ' ' + buildMySqlDataVolume
        removeDockerNetwork(buildNetwork)

        def dockerRegistryHostAndPort = dockerRegistryHost + ':' + dockerRegistryPort
        def remoteImageTagPrefix = dockerRegistryHostAndPort + '/openmpf/'

        def buildImageName = remoteImageTagPrefix + 'openmpf_build:' + imageTag
        def buildContainerId

        def workflowManagerImageName = remoteImageTagPrefix + 'openmpf_workflow_manager:' + imageTag
        def pythonExecutorImageName = remoteImageTagPrefix + 'openmpf_python_executor:' + imageTag

        stage('Clone repos') {
            openmpfDockerSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-docker.git",
                    '.', openmpfDockerBranch)

            // Revert changes made to files by a previous Jenkins build.
            sh 'git reset --hard HEAD '

            def openmpfProjectsPath = 'openmpf_build/openmpf-projects'
            gitCheckoutAndPull("https://github.com/openmpf/openmpf-projects.git",
                    openmpfProjectsPath, openmpfProjectsBranch)
            sh 'cd ' + openmpfProjectsPath + '; git submodule update --init'

            openmpfSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf.git",
                    openmpfProjectsPath + '/openmpf', openmpfBranch)
            openmpfComponentsSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-components.git",
                    openmpfProjectsPath + '/openmpf-components', openmpfComponentsBranch)
            openmpfContribComponentsSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-contrib-components.git",
                    openmpfProjectsPath + '/openmpf-contrib-components', openmpfContribComponentsBranch)
            openmpfCppComponentSdkSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-cpp-component-sdk.git",
                    openmpfProjectsPath + '/openmpf-cpp-component-sdk', openmpfCppComponentSdkBranch)
            openmpfJavaComponentSdkSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-java-component-sdk.git",
                    openmpfProjectsPath + '/openmpf-java-component-sdk', openmpfJavaComponentSdkBranch)
            openmpfPythonComponentSdkSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-python-component-sdk.git",
                    openmpfProjectsPath + '/openmpf-python-component-sdk', openmpfPythonComponentSdkBranch)
            openmpfBuildToolsSha = gitCheckoutAndPull("https://github.com/openmpf/openmpf-build-tools.git",
                    openmpfProjectsPath + '/openmpf-build-tools', openmpfBuildToolsBranch)

            buildShas = 'openmpf-docker: ' + openmpfDockerSha +
                    ', openmpf: ' + openmpfSha +
                    ', openmpf-components: ' + openmpfComponentsSha +
                    ', openmpf-contrib-components: ' + openmpfContribComponentsSha +
                    ', openmpf-cpp-component-sdk: ' + openmpfCppComponentSdkSha +
                    ', openmpf-java-component-sdk: ' + openmpfJavaComponentSdkSha +
                    ', openmpf-python-component-sdk: ' + openmpfPythonComponentSdkSha +
                    ', openmpf-build-tools: ' + openmpfBuildToolsSha

            if (buildCustomComponents) {
                def openmpfCustomDockerPath = 'openmpf_custom_build'
                openmpfCustomDockerSha = gitCheckoutAndPullWithCredId(openmpfCustomDockerRepo, openmpfCustomRepoCredId,
                        openmpfCustomDockerPath, openmpfCustomDockerBranch)

                // Copy custom component build files into place (SDKs, etc.)
                sh 'cp -u /data/openmpf/custom-build-files/* ' + openmpfCustomDockerPath

                def openmpfCustomComponentsPath = openmpfProjectsPath + '/' + openmpfCustomComponentsSlug
                openmpfCustomComponentsSha = gitCheckoutAndPullWithCredId(openmpfCustomComponentsRepo, openmpfCustomRepoCredId,
                        openmpfCustomComponentsPath, openmpfCustomComponentsBranch)

                def openmpfCustomSystemTestsPath = openmpfProjectsPath + '/' + openmpfCustomSystemTestsSlug
                openmpfCustomSystemTestsSha = gitCheckoutAndPullWithCredId(openmpfCustomSystemTestsRepo, openmpfCustomRepoCredId,
                        openmpfCustomSystemTestsPath, openmpfCustomSystemTestsBranch)
            }

            if (applyCustomConfig) {
                def openmpfConfigDockerPath = 'openmpf_custom_config'
                openmpfConfigDockerSha = gitCheckoutAndPullWithCredId(openmpfConfigDockerRepo, openmpfConfigRepoCredId,
                        openmpfConfigDockerPath, openmpfConfigDockerBranch)
            }

            // Copy JDK into place
            sh 'cp -u /data/openmpf/jdk-*-linux-x64.rpm openmpf_build'

            // Copy *package.json into place
            if (buildPackageJson.contains("/")) {
                sh 'cp ' + buildPackageJson + ' ' + openmpfProjectsPath +
                        '/openmpf/trunk/jenkins/scripts/config_files'
                buildPackageJson = buildPackageJson.substring(buildPackageJson.lastIndexOf("/") + 1)
            }

            // Generate compose files
            sh './scripts/docker-generate-compose-files.sh ' + dockerRegistryHost + ':' +
                    dockerRegistryPort + ' openmpf ' + imageTag

            // TODO: Attempt to pull images in separate stage so that they are not
            // built from scratch on a clean Jenkins node.
        }

        docker.withRegistry('http://' + dockerRegistryHostAndPort, dockerRegistryCredId) {

            stage('Build base image') {
                sh 'docker build openmpf_build/' +
                        ' --build-arg BUILD_DATE=' + buildDate +
                        ' --build-arg BUILD_VERSION=' + imageTag +
                        ' --build-arg BUILD_SHAS=\"' + buildShas + '\"' +
                        ' -t ' + buildImageName

                if (buildCustomComponents) {
                    buildShas += ', openmpf-custom-docker: ' + openmpfCustomDockerSha +
                            ', openmpf-custom-components: ' + openmpfCustomComponentsSha +
                            ', openmpf-custom-system-tests: ' + openmpfCustomSystemTestsSha

                    // Build the new build image for custom components using the original build image for open source
                    // components. This overwrites the original build image tag.
                    sh 'docker build openmpf_custom_build/' +
                            ' --build-arg BUILD_IMAGE_NAME=' + buildImageName +
                            ' --build-arg BUILD_DATE=' + buildDate +
                            ' --build-arg BUILD_VERSION=' + imageTag +
                            ' --build-arg BUILD_SHAS=\"' + buildShas + '\"' +
                            ' -t ' + buildImageName
                }
            }

            try {
                stage('Build OpenMPF') {
                    if (!buildOpenmpf) {
                        echo 'SKIPPING OPENMPF BUILD'
                    }
                    when (buildOpenmpf) { // if false, don't show this step in the Stage View UI
                        if (runMvnTests) {
                            sh 'docker network create ' + buildNetwork
                        }

                        // Run container as daemon in background.
                        buildContainerId = sh(script: 'docker run --entrypoint sleep -t -d ' +
                                '--mount type=bind,source=/home/jenkins/.m2,target=/root/.m2 ' +
                                '--mount type=bind,source="$(pwd)"/openmpf_runtime/build_artifacts,target=/mnt/build_artifacts ' +
                                '--mount type=bind,source="$(pwd)"/openmpf_build/openmpf-projects,target=/mnt/openmpf-projects ' +
                                (runMvnTests ? '--mount type=volume,source=' + buildSharedDataVolume +  ',target=/home/mpf/openmpf-projects/openmpf/trunk/install/share ' : '') +
                                (runMvnTests ? '--mount type=bind,source=' + extraTestDataPath + ',target=/mpfdata,readonly ' : '') +
                                (runMvnTests ? '--network=' + buildNetwork +  ' ' : '') +
                                buildImageName + ' infinity', returnStdout: true).trim()

                        sh 'docker exec ' +
                                '-e BUILD_PACKAGE_JSON=' + buildPackageJson + ' ' +
                                buildContainerId + ' /home/mpf/docker-entrypoint.sh'
                    }
                }

                stage('Run Google tests') {
                    if (!runGTests) {
                        echo 'SKIPPING GOOGLE TESTS'
                    }
                    when (runGTests) { // if false, don't show this step in the Stage View UI
                        def gTestsRetval = sh(script: 'docker exec ' +
                                buildContainerId + ' /home/mpf/run-gtests.sh', returnStatus: true)

                        processTestReports()

                        if (gTestsRetval != 0) {
                            sh 'exit ' + gTestsRetval
                        }
                    }
                }

                stage('Build runtime images') {
                    if (!buildRuntimeImages) {
                        echo 'SKIPPING BUILD OF RUNTIME IMAGES'
                    }
                    when (buildRuntimeImages) { // if false, don't show this step in the Stage View UI
                        sh 'docker-compose build' +
                                ' --build-arg BUILD_IMAGE_NAME=' + buildImageName +
                                ' --build-arg BUILD_DATE=' + buildDate +
                                ' --build-arg BUILD_VERSION=' + imageTag +
                                ' --build-arg BUILD_SHAS=\"' + buildShas + '\"'

                        sh 'docker build openmpf_runtime ' +
                                '--file openmpf_runtime/python_executor/Dockerfile ' +
                                "--tag '${pythonExecutorImageName}'"
                    }
                }

                stage('Run Maven tests') {
                    if (!buildOpenmpf || !buildRuntimeImages || !runMvnTests) {
                        echo 'SKIPPING MAVEN TESTS'
                    }
                    when (buildOpenmpf && buildRuntimeImages && runMvnTests) { // if false, don't show this step in the Stage View UI
                        // Add extra test data volume
                        sh 'sed \'/shared_data:\\/opt\\/mpf\\/share/a \\      - ' + extraTestDataPath + ':/mpfdata:ro\'' +
                                ' docker-compose.yml > docker-compose-test.yml'

                        // Update volume and network names
                        sh 'sed -i \'s/shared_data:/' + buildSharedDataVolumeSuffix + ':/g\' docker-compose-test.yml'
                        sh 'sed -i \'s/mysql_data:/' + buildMySqlDataVolumeSuffix + ':/g\' docker-compose-test.yml'
                        sh 'sed -i \'s/compose_overlay/' + buildNetworkSuffix + '/g\' docker-compose-test.yml'

                        // Run supporting containers in background.
                        sh 'docker-compose -f docker-compose-test.yml up -d' +
                                ' --scale workflow_manager=0 --scale node_manager=2'

                        def mvnTestsRetval = sh(script: 'docker exec' +
                                ' -e EXTRA_MVN_OPTIONS=\"' + mvnTestOptions + '\" ' +
                                buildContainerId +
                                ' /home/mpf/run-mvn-tests.sh', returnStatus: true)

                        processTestReports()

                        if (mvnTestsRetval != 0) {
                            sh 'exit ' + mvnTestsRetval
                        }
                    }
                }

            } finally {
                if (buildContainerId != null) {
                    sh 'docker container rm -f ' + buildContainerId

                    if (runMvnTests) {

                        if (fileExists('docker-compose-test.yml')) {
                            sh 'docker-compose -f docker-compose-test.yml rm -svf || true'
                            sh 'sleep 10' // give previous command some time
                        }

                        sh 'docker volume rm -f ' + buildMySqlDataVolume // preserve openmpf_shared_data for post-run analysis
                        removeDockerNetwork(buildNetwork)
                    }
                }
            }

            stage('Apply custom config') {
                if (!applyCustomConfig) {
                    echo 'SKIPPING CUSTOM CONFIGURATION'
                }
                when (applyCustomConfig) { // if false, don't show this step in the Stage View UI
                    buildShas += ', openmpf-config-docker: ' + openmpfConfigDockerSha

                    // Build and tag the new Workflow Manager image with the image tag used in the compose files.
                    // That way, we do not have to modify the compose files. This overwrites the tag that referred
                    // to the original Workflow Manager image without the custom config.
                    sh 'docker build openmpf_custom_config/workflow_manager' +
                            ' --build-arg BUILD_IMAGE_NAME=' + workflowManagerImageName +
                            ' --build-arg BUILD_DATE=' + buildDate +
                            ' --build-arg BUILD_VERSION=' + imageTag +
                            ' --build-arg BUILD_SHAS=\"' + buildShas + '\"' +
                            ' -t ' + workflowManagerImageName
                }
            }

            stage('Push runtime images') {
                if (!pushRuntimeImages) {
                    echo 'SKIPPING PUSH OF RUNTIME IMAGES'
                }
                when (pushRuntimeImages) { // if false, don't show this step in the Stage View UI
                    // Pushing multiple tags is cheap, as all the layers are reused.
                    sh 'docker push ' + buildImageName
                    sh 'docker-compose push'
                    sh "docker push '${pythonExecutorImageName}'"
                }
            }

        } // end docker.withRegistry()
    } catch (Exception e) {
        buildException = e
    }

    def buildStatus
    if (isAborted()) {
        echo 'DETECTED BUILD ABORTED'
        buildStatus = "aborted"
    } else {
        if (buildException != null) {
            echo 'DETECTED BUILD FAILURE'
            echo 'Exception type: ' + buildException.getClass()
            echo 'Exception message: ' + buildException.getMessage()
            buildStatus = "failure"
        } else {
            echo 'DETECTED BUILD COMPLETED'
            echo "CURRENT BUILD RESULT: ${currentBuild.currentResult}"
            buildStatus = currentBuild.currentResult.equals("SUCCESS") ? "success" : "failure"
        }
        // Post build status
        if (postOpenmpfDockerBuildStatus) {
            postBuildStatus("openmpf-docker", openmpfDockerBranch, openmpfDockerSha, buildStatus, githubAuthToken)
        }
        postBuildStatus("openmpf", openmpfBranch, openmpfSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-components", openmpfComponentsBranch, openmpfComponentsSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-contrib-components", openmpfContribComponentsBranch, openmpfContribComponentsSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-cpp-components-sdk", openmpfCppComponentSdkBranch, openmpfCppComponentSdkSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-java-components-sdk", openmpfJavaComponentSdkBranch, openmpfJavaComponentSdkSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-python-components-sdk", openmpfPythonComponentSdkBranch, openmpfPythonComponentSdkSha, buildStatus, githubAuthToken)
        postBuildStatus("openmpf-build-tools", openmpfBuildToolsBranch, openmpfBuildToolsSha, buildStatus, githubAuthToken)
    }

    email(buildStatus, emailRecipients)

    if (buildException != null) {
        throw buildException // rethrow so Jenkins knows of failure
    }
}}

def gitCheckoutAndPull(String repo, String dir, String branch) {
    // This is the official procedure, but we don't want all of the "Git Build Data"
    // entries clogging up the sidebar in the build UI:
    // checkout([$class: 'GitSCM',
    //    branches: [[name: '*/' + branch]],
    //    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: dir]],
    //    userRemoteConfigs: [[url: repo]]])

    if (!branch.isEmpty()) {
        if (!fileExists(dir + '/.git')) {
            sh 'git clone ' + repo + ' ' + dir
        }
        sh 'cd ' + dir + '; git fetch'
        sh 'cd ' + dir + '; git checkout ' + branch
        sh 'cd ' + dir + '; git pull origin ' + branch
    }

    return getGitCommitSha(dir) // assume the repo is already cloned
}

def gitCheckoutAndPullWithCredId(String repo, String credId, String dir, String branch) {
    if (!branch.isEmpty()) {
        def scmVars = checkout([$class: 'GitSCM',
                  branches: [[name: '*/' + branch]],
                  extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: dir]],
                  userRemoteConfigs: [[credentialsId: credId, url: repo]]])

        // TODO: Make sure we're not in a detached state.
        // sh 'cd ' + dir + '; git checkout ' + branch

        return scmVars.GIT_COMMIT
    }

    return getGitCommitSha(dir) // assume the repo is already cloned
}

def getGitCommitSha(String dir) {
    return sh(script: 'cd ' + dir + '; git rev-parse HEAD', returnStdout: true).trim()
}

def isAborted() {
    def actions = currentBuild.getRawBuild().getActions(jenkins.model.InterruptedBuildAction)
    return !actions.isEmpty()
}

def email(String status, String recipients) {
    emailext (
            subject: status.toUpperCase() + ": ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
            // mimeType: 'text/html',
            // body: "<p>Check console output at <a href=\"${env.BUILD_URL}\">${env.BUILD_URL}</a></p>",
            body: '${JELLY_SCRIPT,template="text"}',
            recipientProviders: [[$class: 'RequesterRecipientProvider']],
            to: recipients
    )
}

def getTimestamp() {
    return sh(script: 'date --iso-8601=seconds', returnStdout: true).trim()
}

// TODO: Don't use sudo.
def processTestReports() {
    def newReportsPath = 'openmpf_runtime/build_artifacts/reports'
    def processedReportsPath = newReportsPath + '/processed'

    // Touch files to avoid the following error if the test reports are more than 3 seconds old:
    // "Test reports were found but none of them are new"
    sh 'sudo touch ' + newReportsPath + '/*-reports/*.xml'

    junit newReportsPath + '/*-reports/*.xml'

    sh 'sudo mkdir -p ' + processedReportsPath
    sh 'sudo mv ' + newReportsPath + '/*-reports' + ' ' + processedReportsPath
}

def postBuildStatus(String repo, String branch, String sha, String status, authToken) {
    if (branch.isEmpty()) {
        return
    }

    def resultJson = sh(script: 'echo \'{"state": "' + status + '", ' +
            '"description": "' + currentBuild.projectName + ' ' + currentBuild.displayName + '", ' +
            '"context": "jenkins"}\' | ' +
            'curl -s -X POST -H "Authorization: token ' + authToken + '" ' +
            '-d @- https://api.github.com/repos/openmpf/' + repo + '/statuses/' + sha, returnStdout: true)

    def success = resultJson.contains("\"state\": \"" + status + "\"") &&
            resultJson.contains("\"description\": \"" + currentBuild.projectName + ' ' + currentBuild.displayName + "\"") &&
            resultJson.contains("\"context\": \"jenkins\"")

    if (!success) {
        echo 'Failed to post build status:'
        echo resultJson
    }
}

def removeDockerNetwork(network) {
    if (sh(script: 'docker network inspect ' + network + ' > /dev/null 2>&1', returnStatus: true) == 0) {
        sh 'docker network rm ' + network
    }
}