# Part 1 - Deployment and CI/CD Design

Service: `sync-service`

Platform: Jenkins, Docker, MongoDB, and GCP VMs

---

## Scope

This document describes the CI/CD design for the Spring Boot service named
`sync-service`.

The service connects to MongoDB and is deployed to GCP virtual machines across
three environments:

- `qa`
- `staging`
- `prod`

The design focuses on branching, Jenkins pipeline behavior, configuration
management, deployment, and rollback.

---

## Branching Strategy

The repository uses short-lived work branches and stable environment branches.

- `feature/*`
  - Used for new feature work.
  - Jenkins runs CI validation only.
  - No environment deployment happens from this branch.

- `bugfix/*`
  - Used for bug fixes before merge.
  - Jenkins runs CI validation only.
  - No environment deployment happens from this branch.

- `develop`
  - Used as the integration branch.
  - A successful merge to this branch deploys to `qa`.

- `release/*`
  - Used for release candidates.
  - A successful merge or update deploys to `staging`.

- `main`
  - Used for production-ready code.
  - The pipeline runs automatically after merge.
  - Production deployment requires manual approval.

Branch to environment mapping:

- `feature/*` maps to no environment.
- `bugfix/*` maps to no environment.
- `develop` maps to `qa`.
- `release/*` maps to `staging`.
- `main` maps to `prod`.

---

## Avoiding Accidental Production Deployments

Production deployment is protected by branch rules and manual approval.

Only the `main` branch is eligible for production deployment. Branches such as
`feature/*`, `bugfix/*`, `develop`, and `release/*` cannot deploy to
production.

Even when code is merged into `main`, Jenkins does not deploy to production
immediately. The pipeline first runs validation stages such as build, tests,
Docker image creation, and security scanning.

Before production deployment, Jenkins pauses at a manual approval step. This
approval should be limited to authorized users such as a release manager,
DevOps engineer, or tech lead using Jenkins role-based access control.

The recommended production behavior is:

- Automatically run the pipeline on merge to `main`.
- Build, test, scan, and publish a versioned Docker image.
- Pause before the production deployment stage.
- Deploy to production only after manual approval.

---

## Jenkins Pipeline

The Jenkins pipeline is implemented as a declarative `Jenkinsfile`.

The pipeline first resolves the branch name and determines whether the run is
a PR build, a CI-only branch build, or a deployable branch build.

High-level stages:

1. Checkout source code.
2. Resolve branch context and target environment.
3. Validate branch policy.
4. Handle PR status gate.
5. Compile the Spring Boot service.
6. Run unit tests.
7. Package the application.
8. Build the Docker image.
9. Run image security scan.
10. Push image for deployable branches.
11. Read the previous stable image tag.
12. Wait for production approval, if target is `prod`.
13. Deploy with a rolling deployment strategy.
14. Run health checks.
15. Mark the image as stable after success.
16. Roll back on deployment failure.

Jenkins should be configured as a Multibranch Pipeline. A webhook from the Git
provider should trigger the job on branch updates and merges.

Suggested branch discovery pattern:

```text
^(feature|bugfix)/.*$|^develop$|^release/.+$|^main$
```

What the `Jenkinsfile` does by build type:

- PR build
  - Checks out the code.
  - Resolves branch context.
  - Validates the branch policy.
  - Runs the PR status gate.
  - Does not compile, package, push, or deploy.

- `feature/*` or `bugfix/*` branch push
  - Checks out the code.
  - Compiles the service.
  - Runs unit tests.
  - Packages the Spring Boot artifact.
  - Builds the Docker image.
  - Runs the image security scan.
  - Does not push the image.
  - Does not deploy to any environment.

- `develop` branch run
  - Runs the same build and validation stages.
  - Pushes the Docker image.
  - Deploys to `qa` using rolling deployment.
  - Runs the health check.
  - Marks the image as stable after success.

- `release/*` branch run
  - Runs the same build and validation stages.
  - Pushes the Docker image.
  - Deploys to `staging` using rolling deployment.
  - Runs the health check.
  - Marks the image as stable after success.

- `main` branch run
  - Runs the same build and validation stages.
  - Pushes the Docker image.
  - Reads the previous stable production image.
  - Waits for manual production approval.
  - Deploys to `prod` using rolling deployment.
  - Runs the health check.
  - Marks the image as stable after success.

---

## PR vs Merge Behavior

Every push to a `feature/*` or `bugfix/*` branch triggers CI validation. This
validates the latest commit before it is merged.

The CI validation includes:

- Compile
- Unit tests
- Package
- Docker image build
- Security scan

Pull requests do not deploy to any environment. The PR acts as a merge gate.
The Git provider should require the latest Jenkins branch CI status to be
green before allowing the PR to merge.

In this design, the PR pipeline itself is intentionally lightweight. It does
not rerun compile, test, package, Docker build, or deployment stages. Those
checks already run on each push to the source branch.

This avoids repeating the same expensive pipeline work only because a PR was
opened. If the target branch has changed significantly, Jenkins can optionally
run a lightweight merge-result validation build.

After a PR is merged, Jenkins runs on the target branch:

- Merge to `develop` deploys to `qa`.
- Merge to `release/*` deploys to `staging`.
- Merge to `main` prepares a production deployment.

For `main`, production deployment still waits for manual approval.

---

## Rollback Strategy

Rollback is image-based, not source-code-based.

Each Docker image is tagged with an immutable commit-based tag. Each
environment stores the last successfully deployed image as its previous stable
version.

During deployment, Jenkins updates the environment and then calls the Spring
Boot health endpoint:

```text
/actuator/health
```

If the deployment fails or the health check does not pass, Jenkins stops the
rollout and redeploys the previous stable image.

For rolling deployment, only updated VMs need rollback. VMs that were not yet
updated continue serving the previous stable version.

Rollback steps:

1. Stop the current rollout.
2. Read the previous stable image tag for the environment.
3. Redeploy the previous image to affected VMs.
4. Run the health check again.
5. Mark the Jenkins build as failed.
6. Notify the team with failed and restored image tags.

Production rollback does not require a new code merge. Jenkins restores the
last known-good artifact directly.

---

## Configuration Management

The same Docker image should be promoted across `qa`, `staging`, and `prod`.
Environment differences are handled through runtime configuration.

Spring Boot profiles are used to select environment behavior:

```text
SPRING_PROFILES_ACTIVE=qa
SPRING_PROFILES_ACTIVE=staging
SPRING_PROFILES_ACTIVE=prod
```

Non-sensitive environment configuration can be managed through Jenkins
environment variables or Jenkins parameters.

Examples:

- `APP_ENV`
- `SPRING_PROFILES_ACTIVE`
- `MONGO_DATABASE`
- `LOG_LEVEL`
- `SERVICE_PORT`

Sensitive values must not be committed to Git or written in plain text in the
`Jenkinsfile`.

Secrets are stored in Jenkins Credentials and injected only when required.

Examples:

- MongoDB username
- MongoDB password
- API keys
- Docker registry credentials
- GCP service account credentials

In a later GCP-native setup, runtime secrets can move to GCP Secret Manager.
VMs can access those secrets through IAM instead of long-lived static keys.

---

## Deployment Strategy

The recommended deployment strategy is rolling deployment.

The service should run on multiple GCP VMs behind a load balancer. Jenkins
updates one VM at a time and waits for the health check to pass before moving
to the next VM.

Rolling deployment flow:

1. Remove one VM from load balancer rotation.
2. Deploy the new Docker image to that VM.
3. Start the container with the correct Spring profile.
4. Check `/actuator/health`.
5. Add the VM back to load balancer rotation.
6. Repeat for the next VM.

Rolling deployment gives minimal downtime because healthy old instances keep
serving traffic while the new version is deployed gradually.

Recreate deployment is not preferred because it stops the old version before
the new version is ready. This can cause downtime.

Blue/Green deployment is safer for critical releases, but it requires two
complete production environments. For a startup-style setup, that increases
cost and operational complexity.

Rolling deployment provides the best balance of reliability, cost, and
simplicity for this VM-based service.

---

## Jenkins Credentials

The pipeline expects Jenkins credentials to be created outside the
`Jenkinsfile`.

Required credential examples:

- `gcp-artifact-registry-writer`
- `gcp-vm-deployer-qa`
- `gcp-vm-deployer-staging`
- `gcp-vm-deployer-prod`
- `mongo-qa`
- `mongo-staging`
- `mongo-prod`
- `sync-api-key-qa`
- `sync-api-key-staging`
- `sync-api-key-prod`

Credential values should never be printed in Jenkins logs.
