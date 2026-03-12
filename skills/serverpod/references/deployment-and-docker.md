---
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Serverpod Deployment Strategies & Best Practices (Deployment Strategies)

## Goal
Implements Serverpod deployment to production environments. Deploying Serverpod is highly flexible. Officially, multiple levels of support are provided, from enterprise-grade cloud Terraform scripts to lightweight community Docker solutions, and even support for Serverless platforms.

## Instructions

### Officially Supported Cloud Deployment (Terraform IaC)
Serverpod officially invested significant effort into creating out-of-the-box infrastructure automation scripts (Infrastructure as Code). If you have an AWS or GCP account (Cloud Engine, not Cloud Run), this is the "flagship" deployment method.

When you run `serverpod create`, the project already includes Terraform configurations for AWS/GCP.

#### GCP Cloud Engine (GCE) + Terraform
*   **Architecture**: Automatically sets up an Autoscaling server cluster (GCE), Load Balancer, Cloud SQL (PostgreSQL), Artifact Registry (Docker registry), and Cloud Storage.
*   **Deployment Flow**: Create a GCP project with a Service Account -> Setup domain DNS -> Setup Secrets in GitHub Actions -> Push code, and GitHub Actions will execute `terraform apply` and deploy the docker image.

#### AWS EC2 + Terraform
*   **Architecture**: Includes an Autoscaling EC2 cluster, Application Load Balancer, RDS (Postgres), S3 buckets, CloudFront, and CodeDeploy. It is capable of true Rolling Updates (zero-downtime updates).
*   **Default Scale**: The default Terraform setup aims to stay as close to the AWS Free Tier as possible to save costs.

### Serverless Deployment (e.g., Google Cloud Run)
Many teams prefer a Serverless architecture, like Google Cloud Run, because it "bills only upon traffic."

**⚠️ Critical Warning: Stateful vs Stateless**
Serverpod was initially designed as a long-running Monolith server, responsible for maintaining database connections, WebSocket Streaming, and background scheduling (Future Calls / Cron jobs).
If you want to throw it onto Cloud Run:
1. **Must be Stateless**: If using Streaming, connections will drop because Containers are randomly destroyed.
2. **Loses Background Task Capability**: Because Cloud Run freezes the CPU when there are no HTTP Requests, Serverpod's `FutureCall` or health checks will fail to execute.
3. **Solution (Role Switching)**: You must add the `--role serverless` parameter to the start command to tell the framework this is a stateless instance. Additionally, another cron service (like Cloud Scheduler) must be set up to ping a machine running in `--role maintenance` mode every minute to execute background tasks.

### General & Self-hosted Docker VPS (Most Popular Approach)
For most small-to-medium projects, packaging Serverpod into a Docker Image and deploying it to a single VPS (like DigitalOcean, Linode) is the most economical and flexible solution.

#### Server Startup Mode (Mode)
Serverpod internally determines the configuration file via yaml files under `config/`.
When deploying online, it is imperative to explicitly provide a parameter to the startup command to read `config/production.yaml`:
```bash
./bin/server --mode production --server-id default
```
> If running in a clustered environment, the `--server-id` of each Instance must be different, unless you are in an environment like AWS Fargate where specifying the ID is difficult.

#### Password & Environment Variable Secrecy
**ABSOLUTELY NEVER write your DB password in plaintext inside `config/production.yaml`**.
Production environment passwords must be passed via the `SERVERPOD_PASSWORDS` environment variable, containing a JSON string:
```json
{"database": "YOUR_STRONG_DB_PASSWORD", "redis": "YOUR_REDIS_PASSWORD"}
```

#### Community Support Tools
If you find writing Docker Compose deployments difficult, the community provides the [serverpod_vps](https://pub.dev/packages/serverpod_vps) package. It utilizes a simple CLI to help you deploy Serverpod and Postgres to a basic Virtual Private Server with one click.

### Deployment Checklist
1. **Database Migrations Applied**: When starting the Container, attach the `--apply-migrations` parameter to ensure the database automatically upgrades upon startup:
   `./bin/server --mode production --apply-migrations`
2. **Expose PostgreSQL IP (If Using Managed DB)**: If using GCR/Cloud Run connecting to an external Cloud SQL, verify that the firewall or Public IP settings are correct.
3. **Enable Redis (If Multiple Machines)**: If your Docker is distributed across >= 2 machines by a Load balancer, you must forcibly enable Redis Distributed Cache to ensure state synchronization.

## Constraints
* Ensure production passwords are never hardcoded and always injected via the `SERVERPOD_PASSWORDS` environment variable.
* Make sure you correctly assign `--role serverless` if you choose to deploy stateless on platforms like Cloud Run.
