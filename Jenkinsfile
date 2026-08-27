// Dex Jenkinsfile
def dockerImageName        = 'amds-dex'
def selfhostedImageName    = 'amds-dex-selfhosted'
def dockerPrefixInternal   = 'cc-nexus.ad.catalogic.us:8083/catalogicsoftware/'
def dockerRegistryUrl      = 'https://cc-nexus.ad.catalogic.us:8083'
def dockerRegistryCredsInt = 'cc-nexus.ad.catalogic.us-docker-registry'
def dockerRegistryCredsExt = 'docker.io-docker-registry'
def deploymentRepoUrl      = 'https://github.com/catalogicsoftware/cloudcasa-deployment.git'
def deploymentRepoBranch   = 'master'
def deploymentRepoDir      = 'cloudcasa-deployment'
def helmchartRepoUrl       = 'https://github.com/catalogicsoftware/cloudcasa-server-helmchart.git'
def helmchartRepoBranch    = 'main'
def helmchartRepoDir       = 'cloudcasa-server-helmchart'
def acrRepoUrl             = 'cloudcasa.azurecr.io'
def acrCredentialsId       = 'cloudcasa-azurecr'

def mainBranch = 'cloudcasa-v2.45.1'

properties([parameters([
    booleanParam(name: 'FORCE_BUILD', defaultValue: false,
        description: 'Build/push the dex image even when BRANCH_NAME is not the configured main branch'),
    booleanParam(name: 'PREPARE_REPO_FORCE_INTEGRATION_SCOPE', defaultValue: false,
        description: 'Update the deployment repo integration env even when BRANCH_NAME is not the configured main branch'),
    booleanParam(name: 'PREPARE_REPO_DRY_RUN', defaultValue: false,
        description: 'Show deployment repo diff without committing/pushing'),
])])

def isPullRequest = env.CHANGE_ID as boolean
if (isPullRequest) {
    properties([disableConcurrentBuilds(abortPrevious: true)])
}

node('cloudcasa-build') {
    stage('Checkout') {
        cleanWs()
        checkout scm
    }

    def isMainBranch      = env.BRANCH_NAME == mainBranch
    def buildImage         = isMainBranch || isPullRequest || (params.FORCE_BUILD?.toBoolean())
    def runPrepareRepo      = isMainBranch || (params.PREPARE_REPO_FORCE_INTEGRATION_SCOPE?.toBoolean())
    def prepareRepoDryRun   = params.PREPARE_REPO_DRY_RUN?.toBoolean()

    // e.g. "cloudcasa-v2.45.1" -> "2.45.1"; falls back to the sanitized branch name if it
    // doesn't follow the cloudcasa-vX.Y.Z convention (manual/forced builds off other branches).
    def dexVersionMatch = (env.BRANCH_NAME ?: '') =~ /^cloudcasa-v(.+)$/
    def dexVersion = dexVersionMatch.find() ? dexVersionMatch.group(1) : (env.BRANCH_NAME ?: 'unknown').replaceAll('[^0-9A-Za-z.-]', '-')
    def dockerTag = "${dexVersion}-${env.BUILD_NUMBER}"

    if (!buildImage) {
        echo "Skipping build (branch=${env.BRANCH_NAME})"
        return
    }

    def internalImage = "${dockerPrefixInternal}${dockerImageName}:${dockerTag}"

    stage('Docker build') {
        sh "docker build --build-arg VERSION=${dockerTag} --build-arg BUILDPLATFORM=linux/amd64 -t ${internalImage} ."
    }

    stage('Push to internal registry') {
        docker.withRegistry(dockerRegistryUrl, dockerRegistryCredsInt) {
            sh "docker push ${internalImage}"
        }
    }

    if (isPullRequest) {
        echo "PR build complete. Skipping downstream registry/deployment stages."
        return
    }

    def externalImage = "catalogicsoftware/${dockerImageName}:${dockerTag}"
    stage('Push to external registry') {
        sh "docker tag ${internalImage} ${externalImage}"
        docker.withRegistry('', dockerRegistryCredsExt) {
            sh "docker push ${externalImage}"
        }
    }

    stage('Push selfhosted to Azure CR') {
        def nexusHost = dockerRegistryUrl.replace('https://', '')
        def acrImage = "${acrRepoUrl}/catalogicsoftware/${selfhostedImageName}:${dockerTag}"
        withCredentials([
            usernamePassword(credentialsId: dockerRegistryCredsInt, usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS'),
            usernamePassword(credentialsId: acrCredentialsId, usernameVariable: 'ACR_USER', passwordVariable: 'ACR_PASS')
        ]) {
            sh """
                docker login -u \${NEXUS_USER} -p \${NEXUS_PASS} ${nexusHost}
                docker login -u \${ACR_USER} -p \${ACR_PASS} ${acrRepoUrl}
                docker buildx imagetools create --tag ${acrImage} ${internalImage}
            """
        }
    }

    stage('Update cloudcasa-server-helmchart (selfhosted)') {
        if (!isMainBranch) {
            echo "Skipping cloudcasa-server-helmchart update (branch=${env.BRANCH_NAME})"
            return
        }

        echo "Updating cloudcasa-server-helmchart dexTag to ${dockerTag}"

        withCredentials([usernamePassword(
            credentialsId: 'github-access-token',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'
        )]) {
            dir(helmchartRepoDir) {
                git url: helmchartRepoUrl,
                    branch: helmchartRepoBranch,
                    credentialsId: 'github-access-token'

                sh """
                    git config user.name  "cloudcasabot"
                    git config user.email "cloudcasabot@catalogicsoftware.com"
                """

                sh """
                    set -eu
                    sed -i -E "s/^([[:space:]]*)dexTag:.*/\\1dexTag: ${dockerTag}/" values.yaml

                    if [ -n "\$(git status --porcelain)" ]; then
                        git add -A
                        git commit -m "ci: update dex selfhosted tag to ${dockerTag} [${env.JOB_NAME} #${env.BUILD_NUMBER}]"
                        git pull --rebase https://\${GIT_USER}:\${GIT_PASS}@github.com/catalogicsoftware/cloudcasa-server-helmchart.git ${helmchartRepoBranch}
                        git push https://\${GIT_USER}:\${GIT_PASS}@github.com/catalogicsoftware/cloudcasa-server-helmchart.git ${helmchartRepoBranch}
                    else
                        echo "No cloudcasa-server-helmchart changes detected"
                    fi
                """
            }
        }
    }

    // Production deployment (archimedes/prod/) is intentionally NOT updated automatically here.
    // Only integration env is wired up.
    stage('Prepare deployment repo (integration)') {
        if (!runPrepareRepo) {
            echo "Skipping deployment repo update (branch=${env.BRANCH_NAME}, forceIntegrationScope=${runPrepareRepo})"
            return
        }

        echo "Updating cloudcasa-deployment integration with ${dockerImageName}:${dockerTag}"

        withCredentials([usernamePassword(
            credentialsId: 'github-access-token',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'
        )]) {
            dir(deploymentRepoDir) {
                git url: deploymentRepoUrl,
                    branch: deploymentRepoBranch,
                    credentialsId: 'github-access-token'

                sh """
                    git config user.name  "cloudcasabot"
                    git config user.email "cloudcasabot@catalogicsoftware.com"
                """

                sh """
                    set -eu
                    if [ -d archimedes/integration ]; then
                        matched_files=\$(grep -rl --exclude-dir=.git "${dockerImageName}:" archimedes/integration || true)
                        if [ -n "\${matched_files}" ]; then
                            for f in \${matched_files}; do
                                sed -Ei 's#(catalogicsoftware/${dockerImageName}:)[^"'"'"'[:space:]]+#\\1${dockerTag}#g' "\${f}"
                                sed -Ei 's#(^|[^/A-Za-z0-9_.-])(${dockerImageName}:)[^"'"'"'[:space:]]+#\\1\\2${dockerTag}#g' "\${f}"
                            done
                        fi

                        kustomization_file="archimedes/integration/kustomization.yaml"
                        if [ -f "\${kustomization_file}" ]; then
                            sed -i "/newName: catalogicsoftware\\/${dockerImageName}\$/{n;s/newTag:.*/newTag: ${dockerTag}/}" "\${kustomization_file}"
                        fi
                    fi
                """

                sh """
                    set -eu
                    if [ -n "\$(git status --porcelain)" ]; then
                        if [ "${prepareRepoDryRun}" = "true" ]; then
                            echo "DRY RUN: changes detected, skipping commit/push"
                            git diff --patch > deployment-repo-dry-run.patch
                        else
                            git add -A
                            git commit -m "ci: update ${dockerImageName} to ${dockerTag} [${env.JOB_NAME} #${env.BUILD_NUMBER}]"
                            git pull --rebase https://\${GIT_USER}:\${GIT_PASS}@github.com/catalogicsoftware/cloudcasa-deployment.git ${deploymentRepoBranch}
                            git push https://\${GIT_USER}:\${GIT_PASS}@github.com/catalogicsoftware/cloudcasa-deployment.git ${deploymentRepoBranch}
                        fi
                    else
                        echo "No deployment repo changes detected"
                    fi
                """

                if (prepareRepoDryRun) {
                    archiveArtifacts artifacts: 'deployment-repo-dry-run.patch', allowEmptyArchive: true, onlyIfSuccessful: true
                }
            }
        }
    }
}
