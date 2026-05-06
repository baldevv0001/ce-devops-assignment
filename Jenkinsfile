// Return the branch name from PR, multibranch, or Git metadata.
String normalizeBranchName() {
    if (env.CHANGE_BRANCH) {
        return env.CHANGE_BRANCH
    }
    if (env.BRANCH_NAME) {
        return env.BRANCH_NAME
    }
    if (env.GIT_BRANCH) {
        return env.GIT_BRANCH.replaceFirst('origin/', '')
    }
    return 'unknown'
}

// Map deployable branches to their target environments.
String resolveDeployEnvironment(String branch, String isPrBuild) {
    if (isPrBuild == 'true') {
        return env.NO_DEPLOY_ENV
    }
    if (branch == env.DEVELOP_BRANCH) {
        return env.QA_ENV
    }
    if (branch.startsWith(env.RELEASE_BRANCH_PREFIX)) {
        return env.STAGING_ENV
    }
    if (branch == env.MAIN_BRANCH) {
        return env.PROD_ENV
    }
    return env.NO_DEPLOY_ENV
}

// Build a short immutable tag from the current commit.
String shortCommit() {
    String commit = env.GIT_COMMIT
    if (!commit?.trim()) {
        commit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    }
    return commit.take(12)
}

// Construct the full Artifact Registry image URI.
String imageUri() {
    return "${env.REGISTRY_HOST}/${env.GCP_PROJECT_ID}/" +
        "${env.IMAGE_REPOSITORY}/${env.APP_NAME}:${env.IMAGE_TAG}"
}

// Deploy only non-PR builds mapped to real environments.
Boolean shouldDeploy() {
    return env.IS_PR_BUILD != 'true' && env.DEPLOY_ENV != env.NO_DEPLOY_ENV
}

// Roll back only when there is a previous stable image to restore.
Boolean shouldRollback() {
    return shouldDeploy() && env.PREVIOUS_IMAGE_URI?.trim()
}

pipeline {
    // Run the pipeline on any available Jenkins agent.
    agent any

    // Keep builds traceable, ordered, and limited in history.
    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        skipDefaultCheckout(true)
    }

    // Define branch policy, shared config, and runtime pipeline state.
    environment {
        // Branch filters for automatic multibranch discovery.
        BRANCH_DISCOVERY_REGEX = '^(feature|bugfix)/.*$|^develop$|^release/.+$|^main$'
        RELEASE_BRANCH_PREFIX = 'release/'
        DEVELOP_BRANCH = 'develop'
        MAIN_BRANCH = 'main'

        NO_DEPLOY_ENV = 'none'
        QA_ENV = 'qa'
        STAGING_ENV = 'staging'
        PROD_ENV = 'prod'

        APP_NAME = 'sync-service'
        REGISTRY_HOST = 'us-central1-docker.pkg.dev'
        GCP_PROJECT_ID = 'replace-with-gcp-project'
        IMAGE_REPOSITORY = 'backend-services'
        RELEASE_STATE_BUCKET = 'replace-with-release-state-bucket'
        HEALTH_ENDPOINT = '/actuator/health'

        ARTIFACT_REGISTRY_CREDENTIALS = 'gcp-artifact-registry-writer'
        VM_DEPLOYER_CREDENTIAL_PREFIX = 'gcp-vm-deployer'
        MONGO_CREDENTIAL_PREFIX = 'mongo'
        SYNC_API_KEY_PREFIX = 'sync-api-key'
        DEPLOY_SCRIPT = './scripts/deploy-rolling.sh'
        HEALTH_CHECK_SCRIPT = './scripts/check-health.sh'
        STABLE_IMAGE_FILE = 'stable-image.txt'

        SOURCE_BRANCH = ''
        DEPLOY_ENV = 'none'
        IMAGE_TAG = ''
        IMAGE_URI = ''
        PREVIOUS_IMAGE_URI = ''
        IS_PR_BUILD = 'false'
    }

    // Execute validation, artifact creation, deployment, and rollback flow.
    stages {
        stage('Checkout') {
            // Pull source code from the configured multibranch SCM.
            steps {
                checkout scm
            }
        }

        stage('Resolve Branch Context') {
            // Decide whether this run is CI-only or deployable.
            steps {
                script {
                    env.SOURCE_BRANCH = normalizeBranchName()
                    env.IS_PR_BUILD = env.CHANGE_ID ? 'true' : 'false'
                    env.DEPLOY_ENV = resolveDeployEnvironment(
                        env.SOURCE_BRANCH,
                        env.IS_PR_BUILD
                    )
                    env.IMAGE_TAG = shortCommit()
                    env.IMAGE_URI = imageUri()

                    echo "Source branch: ${env.SOURCE_BRANCH}"
                    echo "Pull request build: ${env.IS_PR_BUILD}"
                    echo "Deployment environment: ${env.DEPLOY_ENV}"
                    echo "Branch regex: ${env.BRANCH_DISCOVERY_REGEX}"
                    echo "Image URI: ${env.IMAGE_URI}"
                }
            }
        }

        stage('Validate Branch Policy') {
            // Stop unsupported branches before any expensive work starts.
            steps {
                script {
                    if (!(env.SOURCE_BRANCH ==~ env.BRANCH_DISCOVERY_REGEX)) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Unsupported branch: ${env.SOURCE_BRANCH}")
                    }
                }
            }
        }

        stage('PR Status Gate') {
            // Treat pull requests as validation gates without deployment.
            when {
                expression { env.IS_PR_BUILD == 'true' }
            }
            steps {
                echo 'PR builds are validation gates only.'
                echo 'Required checks verify the latest branch CI result.'
                echo 'No image push or environment deployment runs on PRs.'
            }
        }

        stage('Compile') {
            // Compile the Spring Boot service on non-PR branch builds.
            when {
                expression { env.IS_PR_BUILD != 'true' }
            }
            steps {
                sh 'mvn -B -DskipTests clean compile'
            }
        }

        stage('Unit Test') {
            // Run automated tests before packaging or deployment.
            when {
                expression { env.IS_PR_BUILD != 'true' }
            }
            steps {
                sh 'mvn -B test'
            }
        }

        stage('Package') {
            // Build the deployable application artifact.
            when {
                expression { env.IS_PR_BUILD != 'true' }
            }
            steps {
                sh 'mvn -B -DskipTests package'
            }
        }

        stage('Docker Build') {
            // Create an immutable container image for this commit.
            when {
                expression { env.IS_PR_BUILD != 'true' }
            }
            steps {
                sh 'docker build --pull -t "$IMAGE_URI" .'
            }
        }

        stage('Security Scan') {
            // Fail the build when the image has high-risk findings.
            when {
                expression { env.IS_PR_BUILD != 'true' }
            }
            steps {
                sh '''
                    trivy image \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      "$IMAGE_URI"
                '''
            }
        }

        stage('Push Image') {
            // Publish the image only for branches mapped to environments.
            when {
                expression { shouldDeploy() }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: env.ARTIFACT_REGISTRY_CREDENTIALS,
                        variable: 'GCP_KEY_FILE'
                    )
                ]) {
                    sh '''
                        gcloud auth activate-service-account \
                          --key-file "$GCP_KEY_FILE"
                        gcloud auth configure-docker \
                          "$REGISTRY_HOST" \
                          --quiet
                        docker push "$IMAGE_URI"
                    '''
                }
            }
        }

        stage('Read Previous Stable Image') {
            // Load the last known-good image for rollback safety.
            when {
                expression { shouldDeploy() }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: env.ARTIFACT_REGISTRY_CREDENTIALS,
                        variable: 'GCP_KEY_FILE'
                    )
                ]) {
                    sh '''
                        gcloud auth activate-service-account \
                          --key-file "$GCP_KEY_FILE"
                        STATE_PATH="$DEPLOY_ENV/$STABLE_IMAGE_FILE"
                        gsutil cat \
                          "gs://$RELEASE_STATE_BUCKET/$STATE_PATH" \
                          > previous-image.txt || true
                    '''
                }
                script {
                    env.PREVIOUS_IMAGE_URI = readFile(
                        'previous-image.txt'
                    ).trim()
                }
            }
        }

        stage('Approve Production Deployment') {
            // Require human approval before changing production.
            when {
                expression { env.DEPLOY_ENV == env.PROD_ENV }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Deploy ${env.IMAGE_URI} to production?",
                        ok: 'Deploy'
                    )
                }
            }
        }

        stage('Rolling Deploy') {
            // Deploy one VM at a time while healthy VMs keep serving traffic.
            when {
                expression { shouldDeploy() }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: "${env.VM_DEPLOYER_CREDENTIAL_PREFIX}-${env.DEPLOY_ENV}",
                        variable: 'GCP_KEY_FILE'
                    ),
                    usernamePassword(
                        credentialsId: "${env.MONGO_CREDENTIAL_PREFIX}-${env.DEPLOY_ENV}",
                        usernameVariable: 'MONGO_USERNAME',
                        passwordVariable: 'MONGO_PASSWORD'
                    ),
                    string(
                        credentialsId: "${env.SYNC_API_KEY_PREFIX}-${env.DEPLOY_ENV}",
                        variable: 'SYNC_API_KEY'
                    )
                ]) {
                    sh '''
                        gcloud auth activate-service-account \
                          --key-file "$GCP_KEY_FILE"
                        export SPRING_PROFILES_ACTIVE="$DEPLOY_ENV"
                        "$DEPLOY_SCRIPT" \
                          --environment "$DEPLOY_ENV" \
                          --image "$IMAGE_URI" \
                          --health-path "$HEALTH_ENDPOINT"
                    '''
                }
            }
        }

        stage('Health Check') {
            // Verify the deployed service before marking it stable.
            when {
                expression { shouldDeploy() }
            }
            steps {
                sh '''
                    "$HEALTH_CHECK_SCRIPT" \
                      --environment "$DEPLOY_ENV" \
                      --health-path "$HEALTH_ENDPOINT"
                '''
            }
        }

        stage('Mark Stable Image') {
            // Save the successful image tag for future rollbacks.
            when {
                expression { shouldDeploy() }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: env.ARTIFACT_REGISTRY_CREDENTIALS,
                        variable: 'GCP_KEY_FILE'
                    )
                ]) {
                    sh '''
                        gcloud auth activate-service-account \
                          --key-file "$GCP_KEY_FILE"
                        STATE_PATH="$DEPLOY_ENV/$STABLE_IMAGE_FILE"
                        printf '%s' "$IMAGE_URI" | gsutil cp - \
                          "gs://$RELEASE_STATE_BUCKET/$STATE_PATH"
                    '''
                }
            }
        }
    }

    // Clean up after every run and roll back failed deployments.
    post {
        failure {
            // Restore the previous stable image when deployment fails.
            script {
                if (shouldRollback()) {
                    echo "Rolling back ${env.DEPLOY_ENV}"
                    echo "Restoring ${env.PREVIOUS_IMAGE_URI}"
                    withCredentials([
                        file(
                            credentialsId: "${env.VM_DEPLOYER_CREDENTIAL_PREFIX}-${env.DEPLOY_ENV}",
                            variable: 'GCP_KEY_FILE'
                        )
                    ]) {
                        sh '''
                            gcloud auth activate-service-account \
                              --key-file "$GCP_KEY_FILE"
                            "$DEPLOY_SCRIPT" \
                              --environment "$DEPLOY_ENV" \
                              --image "$PREVIOUS_IMAGE_URI" \
                              --health-path "$HEALTH_ENDPOINT"
                        '''
                    }
                } else {
                    echo 'Rollback skipped. No deployed environment changed.'
                }
            }
        }
        always {
            deleteDir()
        }
    }
}
