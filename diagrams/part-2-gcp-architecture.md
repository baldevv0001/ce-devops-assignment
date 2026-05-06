# Part 2 - GCP Architecture Diagram

```mermaid
flowchart LR
    user[Users]
    dns[Cloud DNS]
    lb[External App Load Balancer]
    armor[Cloud Armor policy]
    neg[Serverless NEG]
    run[Cloud Run sync-service]
    sa[Runtime Service Account]
    ar[Artifact Registry]
    sm[Secret Manager]
    vpc[Direct VPC egress]
    psc[Private Service Connect]
    atlas[MongoDB Atlas]
    logs[Cloud Logging]
    mon[Cloud Monitoring]
    jenkins[Jenkins]

    user --> dns
    dns --> lb
    armor -. protects .-> lb
    lb --> neg
    neg --> run

    jenkins --> ar
    ar --> run

    run -. uses .-> sa
    sa --> sm
    run --> vpc
    vpc --> psc
    psc --> atlas

    run --> logs
    run --> mon
```

The service is exposed through an external Application Load Balancer. Cloud
Armor is attached as a security policy, and Cloud Run is reached through a
serverless NEG. Direct public access to the default Cloud Run URL is blocked
by ingress settings.

Runtime secrets are read from Secret Manager using the Cloud Run runtime
service account. MongoDB traffic uses Direct VPC egress and Private Service
Connect to reach MongoDB Atlas privately.
