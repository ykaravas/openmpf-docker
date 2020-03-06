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

def imageTag = env.image_tag ?: 'deleteme'
def buildNoCache = env.build_no_cache?.toBoolean() ?: false
def preserveContainersOnFailure = env.preserve_containers_on_failure?.toBoolean() ?: false

def buildCustomComponents = env.build_custom_components?.toBoolean() ?: false
def openmpfCustomRepoCredId = env.openmpf_custom_repo_cred_id
def applyCustomConfig = env.apply_custom_config?.toBoolean() ?: false
def mvnTestOptions = env.mvn_test_options ?: ''

def dockerRegistryHost = env.docker_registry_host
def dockerRegistryPort = env.docker_registry_port
def dockerRegistryPath = env.docker_registry_path ?: "/openmpf"
def dockerRegistryCredId = env.docker_registry_cred_id;
def pushRuntimeImages = env.push_runtime_images?.toBoolean() ?: false

def pollReposAndEndBuild = env.poll_repos_and_end_build?.toBoolean() ?: false

def postBuildStatusEnabled = 'post_build_status' in env ? env.post_build_status.toBoolean() : true
def githubAuthToken = env.github_auth_token
def emailRecipients = env.email_recipients
def extraTestDataPath = env.extra_test_data_path ?: ''


class Repo {
    String name
    String url
    String path
    String branch
    String sha
    String prevSha

    private Repo(path, url, branch, name) {
        this.path = path
        this.url = url;
        this.branch = branch
        this.name = name;
    }

    Repo(path, url, branch) {
        this(path, url, branch, path)
    }

    static Repo projectsSubRepo(name, branch) {
        return new Repo("openmpf-projects/$name", null, branch, name)
    }
}


def openmpfProjectsRepo = new Repo('openmpf-projects', 'https://github.com/openmpf/openmpf-projects.git',
        env.openmpf_projects_branch ?: 'develop')


def openmpfDockerRepo = new Repo('openmpf-docker', 'https://github.com/openmpf/openmpf-docker.git',
        env.openmpf_docker_branch ?: 'develop')


def openmpfRepo = Repo.projectsSubRepo('openmpf', env.openmpf_branch)


def openmpfComponentsRepo = Repo.projectsSubRepo('openmpf-components', env.openmpf_components_branch)

def openmpfContribComponentsRepo = Repo.projectsSubRepo('openmpf-contrib-components',
        env.openmpf_contrib_components_branch)

def openmpfCppSdkRepo = Repo.projectsSubRepo('openmpf-cpp-component-sdk', env.openmpf_cpp_component_sdk_branch)

def openmpfJavaSdkRepo = Repo.projectsSubRepo('openmpf-java-component-sdk', env.openmpf_java_component_sdk_branch)

def openmpfPythonSdkRepo = Repo.projectsSubRepo('openmpf-python-component-sdk',
        env.openmpf_python_component_sdk_branch)

def openmpfBuildToolsRepo = Repo.projectsSubRepo('openmpf-build-tools', env.openmpf_build_tools_branch)


def projectsSubRepos = [ openmpfRepo, openmpfComponentsRepo, openmpfContribComponentsRepo, openmpfCppSdkRepo,
                         openmpfJavaSdkRepo, openmpfPythonSdkRepo, openmpfBuildToolsRepo ]


def customComponentsRepo = new Repo(env.openmpf_custom_components_slug, env.openmpf_custom_components_repo,
        env.openmpf_custom_components_branch ?: 'develop')

def customSystemTestsRepo = new Repo(env.openmpf_custom_system_tests_slug, env.openmpf_custom_system_tests_repo,
        env.openmpf_custom_system_tests_branch ?: 'develop')

def customConfigRepo = new Repo(env.openmpf_config_docker_slug, env.openmpf_config_docker_repo,
        env.openmpf_config_docker_branch ?: 'develop')

def customRepos = []
if (buildCustomComponents) {
    customRepos << customComponentsRepo << customSystemTestsRepo
    if (applyCustomConfig) {
        customRepos << customConfigRepo
    }
}

def allRepos = [openmpfDockerRepo, openmpfProjectsRepo] + projectsSubRepos + customRepos


node(env.jenkins_nodes) {
wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) { // show color in Jenkins console

def buildException
def inProgressTag

try {
    def buildId = "${currentBuild.projectName}_${currentBuild.number}"
    // Use inProgressTag to ensure concurrent builds don't use the same image tag.
    inProgressTag = buildId


    def dockerRegistryHostAndPort = dockerRegistryHost
    if (dockerRegistryPort) {
        dockerRegistryHostAndPort += ':' + dockerRegistryPort
    }

    def remoteImagePrefix = dockerRegistryHostAndPort
    if (dockerRegistryPath) {
        if (!dockerRegistryPath.startsWith("/")) {
            remoteImagePrefix += "/"
        }
        remoteImagePrefix += dockerRegistryPath
        if (!dockerRegistryPath.endsWith("/")) {
            remoteImagePrefix += "/"
        }
    }

    stage('Clone repos') {
        for (repo in allRepos) {
            if (fileExists(repo.path)) {
                repo.prevSha = shOutput "cd $repo.path && git rev-parse HEAD"
            }
            else {
                repo.prevSha = 'NONE'
            }
        }

        if (!fileExists(openmpfProjectsRepo.path)) {
            sh "git clone --recurse-submodules $openmpfProjectsRepo.url"
        }
        dir(openmpfProjectsRepo.path) {
            sh 'git clean -ffd'
            sh 'git submodule foreach git clean -ffd'
            sh 'git fetch --recurse-submodules'
            sh "git checkout 'origin/$openmpfProjectsRepo.branch'"
            sh 'git submodule update --init'
        }
        for (repo in projectsSubRepos) {
            if (repo.branch && !repo.branch.isAllWhitespace()) {
                sh "cd '$repo.path' && git checkout 'origin/$repo.branch'"
            }
        }


        if (!fileExists(openmpfDockerRepo.path)) {
            sh "git clone $openmpfDockerRepo.url"
        }
        dir(openmpfDockerRepo.path) {
            sh 'rm -rf test-reports/*'
            sh 'git clean -ffd'
            sh 'git fetch'
            sh "git checkout 'origin/$openmpfDockerRepo.branch'"
        }

        for (repo in customRepos) {
            checkout(
                    $class: 'GitSCM',
                    userRemoteConfigs: [[url: repo.url, credentialsId: openmpfCustomRepoCredId]],
                    branches: [[name: repo.branch]],
                    extensions: [
                            [$class: 'CleanBeforeCheckout'],
                            [$class: 'RelativeTargetDirectory', relativeTargetDir: repo.path]])
        }

        for (repo in allRepos) {
            repo.sha = shOutput "cd $repo.path && git rev-parse HEAD"
        }
    } // stage('Clone repos')

    optionalStage('Check repos for updates', pollReposAndEndBuild) {
        echo 'CHANGES:'

        def requiresBuild = false

        for (repo in allRepos) {
            requiresBuild |= (repo.prevSha != repo.sha)
            echo "$repo.name: $repo.prevSha --> $repo.sha"
        }
        echo "REQUIRES BUILD: $requiresBuild"
        currentBuild.result = requiresBuild ? 'SUCCESS' : 'ABORTED';
    }

    if (pollReposAndEndBuild) {
        return // end build early; do this outside of a stage
    }

    def componentComposeFiles
    def runtimeComposeFiles

    stage('Build images') {
        // Make sure we are using most recent version of external images
        for (externalImage in ['centos:7', 'webcenter/activemq', 'postgres:alpine', 'redis:alpine']) {
            sh "docker pull '$externalImage'"
        }

        withEnv(['DOCKER_BUILDKIT=1', 'RUN_TESTS=true']) {
            def noCacheArg = buildNoCache ? '--no-cache' : ''
            def commonBuildArgs = " --build-arg BUILD_TAG='$inProgressTag' $noCacheArg "

            dir ('openmpf-docker') {
                sh 'docker build -f openmpf_build/Dockerfile ../openmpf-projects --build-arg RUN_TESTS ' +
                        "$commonBuildArgs -t openmpf_build:$inProgressTag"

                // --no-cache needs to be handled differently for the openmpf_integration_tests image because it
                // expects that openmpf_build will populate the mvn_cache cache mount. When you run a
                // --no-cache build, Docker will clear any cache mounts used in the Dockerfile right before
                // beginning the build. If we were to just do a regular --no-cache build for openmpf_integration_tests,
                // the mvn_cache mount will be empty.
                if (buildNoCache) {
                    // openmpf_integration_tests Dockerfile uses two stages. The final stage uses openmpf_build as the
                    // base image, so the --no-cache build of openmpf_build invalidates the cache for that stage.
                    // In order to invalidate the cache for the first stage (download_dependencies), we do a --no-cache
                    // build with download_dependencies as the target
                    sh 'docker build integration_tests --target download_dependencies --no-cache'
                }
                sh "docker build integration_tests $commonBuildArgs --no-cache=false " +
                        " -t openmpf_integration_tests:$inProgressTag"
            }

            if (buildCustomComponents) {
                sh "docker build $customSystemTestsRepo.path $commonBuildArgs " +
                        " -t openmpf_integration_tests:$inProgressTag "
            }


            dir('openmpf-docker/components') {
                def cppShas = getVcsRefLabelArg([openmpfCppSdkRepo])
                sh "docker build . -f cpp_component_build/Dockerfile $commonBuildArgs $cppShas " +
                        " -t openmpf_cpp_component_build:$inProgressTag"

                sh "docker build . -f cpp_executor/Dockerfile $commonBuildArgs $cppShas " +
                        " -t openmpf_cpp_executor:$inProgressTag"


                def javaShas = getVcsRefLabelArg([openmpfJavaSdkRepo])
                sh "docker build . -f java_component_build/Dockerfile $commonBuildArgs $javaShas " +
                        " -t openmpf_java_component_build:$inProgressTag"

                sh "docker build . -f java_executor/Dockerfile $commonBuildArgs $javaShas " +
                        " -t openmpf_java_executor:$inProgressTag"


                def pythonShas = getVcsRefLabelArg([openmpfPythonSdkRepo])
                sh "docker build . -f python_component_build/Dockerfile $commonBuildArgs $pythonShas " +
                        " -t openmpf_python_component_build:$inProgressTag"

                sh "docker build . -f python_executor/Dockerfile $commonBuildArgs $pythonShas " +
                        " -t openmpf_python_executor:$inProgressTag"
            }

            dir ('openmpf-docker') {
                sh 'cp .env.tpl .env'

                componentComposeFiles = 'docker-compose.components.yml'
                if (buildCustomComponents) {
                    componentComposeFiles += ":../$customComponentsRepo.path/docker-compose.custom-components.yml"
                }
                runtimeComposeFiles = "docker-compose.core.yml:$componentComposeFiles"

                withEnv(["TAG=$inProgressTag", "COMPOSE_FILE=$runtimeComposeFiles", 'COMPOSE_DOCKER_CLI_BUILD=1']) {
                    sh "docker-compose build $commonBuildArgs --build-arg RUN_TESTS --parallel"

                    def composeYaml = readYaml(text: shOutput('docker-compose config'))
                    addVcsRefLabels(composeYaml, openmpfRepo, openmpfDockerRepo)
                }
            }

            if (applyCustomConfig) {
                echo 'APPLYING CUSTOM CONFIGURATION'
                dir(customConfigRepo.path) {
                    def wfmShasArg = getVcsRefLabelArg([openmpfRepo, openmpfDockerRepo, customConfigRepo])
                    sh "docker build workflow_manager $commonBuildArgs $wfmShasArg " +
                            " -t openmpf_workflow_manager:$inProgressTag"

                    def amqShasArg = getVcsRefLabelArg([openmpfDockerRepo, customConfigRepo])
                    sh "docker build activemq $commonBuildArgs $amqShasArg " +
                            " -t openmpf_activemq:$inProgressTag"
                }
            }
            else  {
                echo 'SKIPPING CUSTOM CONFIGURATION'
            }
        } // withEnv
    } // stage('Build images')

    stage('Run Integration Tests') {
        dir('openmpf-docker') {
            def composeFiles = "docker-compose.integration.test.yml:$componentComposeFiles"
            echo "extraTestDataPath: $extraTestDataPath" // DEBUG
            if (extraTestDataPath) {
                echo "Add docker-compose.stress.test.yml" // DEBUG
                composeFiles += ":docker-compose.stress.test.yml"
            }

            def nproc = shOutput('nproc') as int
            def servicesInSystemTests = ['ocv-face-detection', 'darknet-detection', 'dlib-face-detection',
                                        'ocv-dnn-detection', 'oalpr-license-plate-text-detection',
                                        'ocv-person-detection', 'mog-motion-detection', 'subsense-motion-detection']

            def scaleArgs = servicesInSystemTests.collect({ "--scale '$it=$nproc'" }).join(' ')
            // Sphinx uses a huge amount of memory so we don't want more than 2 of them.
            scaleArgs += " --scale sphinx-speech-detection=${Math.min(nproc, 2)} "

            withEnv(["TAG=$inProgressTag",
                     "EXTRA_MVN_OPTIONS=$mvnTestOptions",
                     "EXTRA_TEST_DATA_PATH=$extraTestDataPath",
                     // Use custom project name to allow multiple builds on same machine
                     "COMPOSE_PROJECT_NAME=openmpf_$buildId",
                     "COMPOSE_FILE=$composeFiles"]) {
                try {
                    echo "echo EXTRA_TEST_DATA_PATH: $EXTRA_TEST_DATA_PATH" // DEBUG
                    sh "docker-compose up --exit-code-from workflow-manager $scaleArgs"
                    sh 'docker-compose down --volumes'
                }
                catch (e) {
                    if (preserveContainersOnFailure) {
                        sh 'docker-compose stop'
                    }
                    else {
                        sh 'docker-compose down --volumes'
                    }
                    throw e;
                }
                finally {
                    junit 'test-reports/*-reports/*.xml'
                }
            } // withEnv
        } // dir('openmpf-docker')
    } // stage('Run Integration Tests')

    stage('Re-Tag Images') {
        reTagImages(inProgressTag, remoteImagePrefix, imageTag)
    }

    optionalStage('Push runtime images', pushRuntimeImages) {
        withEnv(["TAG=$imageTag", "REGISTRY=$remoteImagePrefix", "COMPOSE_FILE=$runtimeComposeFiles"]) {

            docker.withRegistry("http://$dockerRegistryHostAndPort", dockerRegistryCredId) {
                sh "docker push '${remoteImagePrefix}openmpf_cpp_component_build:$imageTag'"
                sh "docker push '${remoteImagePrefix}openmpf_cpp_executor:$imageTag'"

                sh "docker push '${remoteImagePrefix}openmpf_java_component_build:$imageTag'"
                sh "docker push '${remoteImagePrefix}openmpf_java_executor:$imageTag'"

                sh "docker push '${remoteImagePrefix}openmpf_python_component_build:$imageTag'"
                sh "docker push '${remoteImagePrefix}openmpf_python_executor:$imageTag'"

                sh 'cd openmpf-docker && docker-compose push'
            } // docker.withRegistry ...
        } // withEnv...
    } // optionalStage('Push runtime images', ...
}
catch (e) { // Global exception handler
    buildException = e
    throw e
}
finally {
    def buildStatus
    if (isAborted()) {
        echo 'DETECTED BUILD ABORTED'
        buildStatus = 'failure'
    }
    else if (buildException != null) {
        echo 'DETECTED BUILD FAILURE'
        echo 'Exception type: ' + buildException.getClass()
        echo 'Exception message: ' + buildException.getMessage()
        buildStatus = 'failure'
    }
    else {
        echo 'DETECTED BUILD COMPLETED'
        echo "CURRENT BUILD RESULT: ${currentBuild.currentResult}"
        buildStatus = currentBuild.currentResult == 'SUCCESS' ? 'success' : 'failure'
    }

    if (buildStatus != 'success') {
        // Re-tag images after failure so we don't end up with a bunch of images for every failed build.
        reTagImages(inProgressTag, '', 'failed-build-deleteme')
    }

    if (postBuildStatusEnabled) {
        postBuildStatus(openmpfDockerRepo, buildStatus, githubAuthToken)
        for (repo in projectsSubRepos) {
            postBuildStatus(repo, buildStatus, githubAuthToken)
        }
    }
    email(buildStatus, emailRecipients)

    dockerCleanUp()
}
} // wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm'])
} // node(env.jenkins_nodes)


def addVcsRefLabels(composeYaml, openmpfRepo, openmpfDockerRepo) {
    def commonVcsRefs = formatVcsRefs([openmpfRepo, openmpfDockerRepo])

    for (def serviceName in composeYaml.services.keySet()) {
        def service = composeYaml.services[serviceName]
        if (!service.build) {
            echo "Not labeling $service.image since we didn't build it"
            continue
        }

        // workflow-manager and markup have their build context within the openmpf-docker repo, but their content
        // is really based on the openmpf repo.
        if (serviceName == 'workflow-manager' || serviceName == 'markup') {
            addLabelToImage(service.image, 'org.label-schema.vcs-ref', commonVcsRefs)
            continue
        }

        def contextDir = service.build.context
        def tld = shOutput "cd '$contextDir' && basename \$(git rev-parse --show-toplevel)"
        def sha = shOutput "cd '$contextDir' && git rev-parse HEAD"

        prependImageLabel(service.image, 'org.label-schema.vcs-ref', "$tld: $sha, $commonVcsRefs")
    }
}


def addLabelToImage(imageName, labelName, labelValue) {
    sh "echo 'FROM $imageName' | docker build - -t $imageName --label '$labelName=$labelValue'"
}

def prependImageLabel(imageName, labelName, labelValue) {
    def existingLabelValue = shOutput(
            /docker image inspect $imageName --format '{{index .Config.Labels "$labelName"}}'/)

    if (existingLabelValue) {
        labelValue += ", $existingLabelValue"
    }
    addLabelToImage(imageName, labelName, labelValue)
}

def formatVcsRefs(repos) {
    return repos.collect { "$it.name: $it.sha" }.join(', ');
}

def getVcsRefLabelArg(repos) {
    def shas = formatVcsRefs(repos)
    return " --label org.label-schema.vcs-ref='$shas'"
}


def isAborted() {
    return currentBuild.result == 'ABORTED' ||
            !currentBuild.getRawBuild().getActions(jenkins.model.InterruptedBuildAction).isEmpty()
}

def postBuildStatus(repo, status, githubAuthToken) {
    if (!repo.branch || repo.branch.isAllWhitespace()) {
        return
    }

    def description = "$currentBuild.projectName $currentBuild.displayName"
    def statusJson = /{ "state": "$status", "description": "$description", "context": "jenkins" }/
    def url = "https://api.github.com/repos/openmpf/$repo.name/statuses/$repo.sha"
    def response = shOutput "curl -s -X POST -H 'Authorization: token $githubAuthToken' -d '$statusJson' $url"

    def resultJson = readJSON(text: response)

    def success = (resultJson.state == status && resultJson.description == description
                    && resultJson.context == "jenkins")
    if (!success) {
        echo 'Failed to post build status:'
        echo response
    }
}

def email(status, recipients) {
    emailext(
        subject: "$status: $env.JOB_NAME [$env.BUILD_NUMBER]",
        body: '${JELLY_SCRIPT,template="text"}',
        recipientProviders: [[$class: 'RequesterRecipientProvider']],
        to: recipients);
}


def reTagImages(inProgressTag, remoteImagePrefix, imageTag) {
    def imageNames = shOutput("docker images 'openmpf_*:$inProgressTag' --format '{{.Repository}}'").split('\n')

    for (def imageName: imageNames) {
        def inProgressName = "$imageName:$inProgressTag"
        def finalName = "${remoteImagePrefix}${imageName}:$imageTag"
        sh "docker tag $inProgressName $finalName"
        // When an image has multiple tags `docker image rm` only removes the specified tag
        sh "docker image rm $inProgressName"
    }
}


def dockerCleanUp() {
    try {
        def daysUntilRemoval = 7
        def hoursUntilRemoval = daysUntilRemoval * 24
        // Remove dangling <none> images that are more than 1 week old.
        sh "docker image prune --force --filter 'until=${hoursUntilRemoval}h'"

        echo "Checking for deleteme images older than $daysUntilRemoval days."

        def images = shOutput("docker images --filter 'dangling=false' --format '{{.Repository}}:{{.Tag}}'")\
                        .split("\n")

        def now = java.time.Instant.now()
        for (image in images) {
            if (!image.contains('deleteme')) {
                continue;
            }

            // Time format from Docker (it includes quotes at beginning and end): "2019-11-18T18:58:33.990718123Z"
            def quotedTagTimeString = shOutput "docker image inspect --format '{{json .Metadata.LastTagTime}}' $image"
            def tagTimeString = quotedTagTimeString[1..-2]
            def tagTime = java.time.Instant.parse(tagTimeString)

            def daysSinceLastTag = tagTime.until(now, java.time.temporal.ChronoUnit.DAYS)
            if (daysSinceLastTag > daysUntilRemoval) {
                echo "Deleting $image because has \"deleteme\" in its name and was last tagged $daysSinceLastTag days ago."
                sh "docker image rm $image"
            }
        }

        sh 'docker builder prune --force --keep-storage=120GB'
    }
    catch (e) {
        echo "Docker clean up failed due to: $e"
    }
}


def shOutput(script) {
    return sh(script: script, returnStdout: true).trim()
}

def optionalStage(name, condition, body) {
    if (condition) {
        stage (name, body);
    }
    else {
        stage(name) {
            echo "SKIPPING STAGE: $name"
            org.jenkinsci.plugins.pipeline.modeldefinition.Utils.markStageSkippedForConditional(name)
        }
    }
}
