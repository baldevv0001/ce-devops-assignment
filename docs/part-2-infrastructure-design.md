# Part 2 - Infrastructure Design

Service: `sync-service`

Platform: Google Cloud Platform

---

## Scope

This document proposes a GCP infrastructure setup for `sync-service` with
auto-scaling, secure access, and reasonable startup cost.

The service is a Spring Boot backend that runs as a container and connects to
MongoDB.

---

## Architecture Overview

The recommended architecture uses:

- Cloud Run for compute.
- Artifact Registry for container images.
- External Application Load Balancer for public HTTPS ingress.
- Cloud Armor for edge security policy.
- Serverless NEG to route load balancer traffic to Cloud Run.
- Direct VPC egress for private outbound access.
- Private Service Connect for MongoDB Atlas private connectivity.
- Secret Manager for runtime secrets.
- Cloud Logging and Cloud Monitoring for observability.

Request flow:

1. Users access the service through a public domain.
2. Cloud DNS points the domain to the Application Load Balancer.
3. The load balancer terminates HTTPS.
4. Cloud Armor applies security and rate-limit policies.
5. The serverless NEG routes traffic to Cloud Run.
6. Cloud Run runs the `sync-service` container.
7. The service connects to MongoDB Atlas through private networking.

Architecture diagram:

- [Part 2 - GCP Architecture Diagram](../diagrams/part-2-gcp-architecture.md)

---

## Compute Choice

Cloud Run is the selected compute option.

Cloud Run is preferred because it runs containers without requiring the team to
manage VMs, node pools, or Kubernetes clusters. It supports automatic scaling
based on request traffic and can scale to zero for low-traffic environments.

This is a good fit for startup constraints because it keeps idle cost and
operational overhead low.

GKE is not selected because it adds cluster cost and operational complexity for
a single backend service. Compute Engine is not selected because it requires
VM patching, instance templates, and more manual scaling setup.

Recommended Cloud Run services:

- `sync-service-qa`
- `sync-service-staging`
- `sync-service-prod`

Each environment should have its own Cloud Run service, service account,
environment variables, and secrets.

---

## Auto-Scaling And Cost Control

Cloud Run should use request-based auto-scaling.

Recommended settings:

- QA: minimum instances `0`, maximum instances `2`.
- Staging: minimum instances `0`, maximum instances `3`.
- Production: minimum instances `1`, if cold starts are not acceptable.

Production maximum instances should be based on expected traffic and MongoDB
connection capacity.

The application should use a bounded MongoDB connection pool so sudden
scale-out does not exhaust database connections.

---

## MongoDB Hosting Approach

MongoDB Atlas on GCP is the recommended MongoDB hosting option.

Atlas is preferred because it provides managed MongoDB with backups,
monitoring, upgrades, and high availability options. This avoids the
operational overhead of running MongoDB on self-managed VMs.

Recommended setup:

- Use MongoDB Atlas in the same GCP region as Cloud Run.
- Use a dedicated production Atlas cluster.
- Use smaller QA and staging clusters to control cost.
- Enable automated backups for production.
- Use Private Service Connect for private production connectivity.

Private Service Connect should be used for production and other dedicated
Atlas clusters. If very small QA or staging tiers do not support private
endpoints, they should still use TLS, strong credentials, and restricted
network access.

---

## Networking Basics

Public ingress should go through an external Application Load Balancer.

The public path is:

```text
Users -> Cloud DNS -> Load Balancer -> Serverless NEG -> Cloud Run
```

Cloud Armor should be attached to the load balancer backend for security rules
and rate limiting.

Cloud Run ingress should be set to:

```text
internal-and-cloud-load-balancing
```

This blocks direct public access to the default Cloud Run URL and allows
traffic through the load balancer.

Private database traffic should use:

```text
Cloud Run -> Direct VPC egress -> Private Service Connect -> MongoDB Atlas
```

If the service needs public internet egress for third-party APIs, it can use
normal egress. If stricter egress control is required, route all egress
through the VPC and use Cloud NAT.

---

## Secrets And IAM

Runtime secrets should be stored in Secret Manager.

Examples:

- MongoDB connection string
- MongoDB username
- MongoDB password
- API keys
- JWT signing secret

Each environment should use a separate Cloud Run runtime service account:

- `sync-service-qa-sa`
- `sync-service-staging-sa`
- `sync-service-prod-sa`

IAM should follow least privilege:

- Grant Secret Manager Secret Accessor only on required secrets.
- Keep deployer permissions separate from runtime permissions.
- Avoid using default service accounts for runtime workloads.
- Grant Artifact Registry read access only where needed.

For cross-project image pulls, grant Artifact Registry Reader to the Cloud Run
service agent for the required repository.

---

## Logging And Monitoring

Use Google Cloud Observability for logs, metrics, and alerts.

Recommended components:

- Cloud Logging for application logs and request logs.
- Cloud Monitoring for metrics, dashboards, and alerts.
- Error Reporting for application exceptions.
- Uptime checks for the health endpoint.
- Log-based metrics for recurring application failures.

Recommended alerts:

- High 5xx error rate.
- High request latency.
- Cloud Run container restart spikes.
- Cloud Run instance count near maximum.
- Health check failures.
- MongoDB connection errors.

The Spring Boot service should expose:

```text
/actuator/health
```

This endpoint can be used by uptime checks and deployment validation.

---

## Deployment And Rollback

Jenkins builds the container image and pushes it to Artifact Registry.

Cloud Run creates a new revision for each deployment. Production deployments
should shift traffic gradually to the new revision instead of moving all
traffic at once.

Rollback is done by moving traffic back to the previous healthy Cloud Run
revision.

---

## Cost And Security Summary

This design keeps cost reasonable by using managed serverless compute.

Cost controls:

- Cloud Run scales to zero for QA and staging.
- Maximum instance limits prevent runaway scale-out.
- GKE cluster overhead is avoided.
- Idle VM cost and VM patching are avoided.
- QA and staging can use smaller MongoDB Atlas tiers.

Security controls:

- HTTPS-only public access.
- Cloud Armor policy on the load balancer backend.
- Cloud Run direct public URL blocked by ingress settings.
- Private MongoDB access through Private Service Connect.
- Secrets stored in Secret Manager.
- Separate service accounts per environment.
- Least-privilege IAM roles.
