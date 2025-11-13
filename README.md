# Keycloak on AWS — Architecture, Dependencies & Complete Setup

This document gives a clear, pragmatic, production-ready picture of running Keycloak on AWS: what servers/services are used, recommended configurations, dependencies (DB, cache, storage, networking), monitoring (CloudWatch / Prometheus / Grafana), HA patterns, security, backups, and a step-by-step deployment plan (Kubernetes/Helm and alternative EC2/ECS options). Use this as a playbook and checklist for implementation.

---

## 1. High-level options (pick one based on your org needs)

* **Kubernetes (recommended):** Run Keycloak on EKS using the official Keycloak Helm chart. Best for scalability, lifecycle, and observability.
* **ECS / Fargate:** Simpler serverless containers without managing nodes. Good if you already use ECS.
* **EC2 (VMs):** Classic approach — Keycloak on EC2 Auto Scaling Group. Gives full control but more ops work.

This doc focuses on **EKS (Helm)** with notes for ECS/EC2 alternatives.

---

## 2. Components (what to run in AWS)

1. **Application tier — Keycloak servers**

   * Run as multiple replicas (3+) for HA
   * Options: Kubernetes Deployment (Stateless) or Stateful for clustered caches if using Infinispan
2. **Load balancer**

   * **ALB** (Application Load Balancer) in front of Keycloak for TLS termination and path-based routing
3. **Database**

   * **Amazon Aurora PostgreSQL (recommended)** or RDS PostgreSQL (Multi-AZ)
   * Single source of truth for realms, users, clients, tokens
4. **Session / cache store**

   * **ElastiCache (Redis)** OR **Infinispan** (if you need distributed cache with advanced features)
   * Keycloak needs a fast cache for sessions and cluster coordination
5. **Object storage**

   * **S3** for storing user upload assets, themes, and backup exports
6. **Secrets management**

   * **AWS Secrets Manager** or **SSM Parameter Store (SecureString)** for DB credentials, cipher secrets
7. **Certificates / DNS**

   * **Route 53** for DNS
   * **AWS Certificate Manager (ACM)** for TLS certs attached to ALB
8. **Monitoring & Logging**

   * **CloudWatch** for logs/metrics and container insights
   * **Prometheus + Grafana** (optional or complementary) for service metrics and richer dashboards
   * **Managed Grafana** (Amazon Managed Grafana) recommended for managed UI/alerts
9. **Tracing (optional)**

   * **AWS X-Ray** or Jaeger if you want distributed traces
10. **Encryption / KMS**

    * **AWS KMS** for envelope encryption of secrets/backups
11. **Backup & DR**

    * Snapshotting Aurora/RDS, S3 lifecycle, Keycloak realm export to S3
12. **IAM & Security**

    * Fine-grained IAM roles for service accounts, encryption keys, S3 access, Secrets Manager access

---

## 3. How many servers (instances/replicas) and configuration

**Kubernetes (EKS) example**

* Keycloak replicas: **3** (minimum for quorum and smooth rolling upgrades)
* Pod resources (starting point per replica):

  * `cpu: 1` / `memory: 2Gi` for lightweight usage
  * `cpu: 2` / `memory: 4Gi` for medium production workloads
  * `liveness/readiness` probes configured
* Node sizes (EKS worker nodes): `m6i.large` or `t3.medium` for small; `m6i.xlarge` for medium load
* Node count: **3–5** (autoscaled) depending on traffic and resource usage

**ECS / EC2**

* EC2 ASG with **3** instances behind ALB, instance types as above.
* Use Auto Scaling and health checks.

**Database sizing**

* Aurora R/W instance `db.r6g.large` as starting point (adjust for throughput)
* Multi-AZ and read replicas if heavy read traffic

**ElastiCache (Redis)**

* Start with a **cluster mode disabled** single primary with replicas, scale to cluster-mode enabled if needed.
* Instance class e.g., `cache.t3.medium`/`cache.m6g.large` for production.

---

## 4. Networking & Security (VPC design)

* Place Keycloak pods in **private subnets**.
* ALB in **public subnets** with security group allowing 443 from internet (or internal only if used by apps behind VPN).
* Security groups:

  * ALB -> Keycloak SG (443 -> 8080/8443)
  * Keycloak SG -> DB SG (worker -> Aurora port 5432)
  * Keycloak SG -> ElastiCache SG (redis port)
* Use **network ACLs** and least-privilege rules.
* Use **IAM OIDC** integration with EKS to map ServiceAccounts to IAM roles (for S3/Secrets access).

---

## 5. Persistence, backups & DR

* DB backups: use **Aurora automated backups** + manual snapshots before upgrades
* Keycloak export: schedule periodic realm/user export to S3 (use `kcadm.sh` or built-in export tools)
* S3: enable versioning and lifecycle policies; optionally cross-region replication for DR
* Redis persistence: enable RDB/AOF or use replicas + snapshots

---

## 6. Logging & Monitoring

### CloudWatch

* **CloudWatch Logs**: send Keycloak logs (stdout from container) to CloudWatch Logs using either:

  * EKS: Container Insights and fluentbit (as DaemonSet) to push logs to CloudWatch Logs
  * ECS: CloudWatch Logs driver
  * EC2: CloudWatch Agent
* **CloudWatch Metrics**:

  * Use Container Insights for pod/cpu/memory metrics
  * Create custom metrics for login rate, failed logins, token issuance rate (Keycloak can expose metrics via Prometheus exporter; push to CloudWatch if required)
* **Alarms**: set alarms on CPU, OOM, error-rate metric, DB connections, and replica lag

### Prometheus & Grafana (recommended for auth metrics)

* Deploy **Prometheus** (kube-prometheus-stack) in EKS or use **Amazon Managed Service for Prometheus (AMP)**.
* Use **Keycloak Prometheus exporter** (community exporter or Keycloak built-in metrics if using Quarkus distribution) to expose metrics such as:

  * `keycloak_requests_total`
  * `keycloak_failed_logins_total`
  * `keycloak_active_sessions`
* **Grafana**: use Amazon Managed Grafana or self-host Grafana. Import/author dashboards for Keycloak + Postgres + JVM/Pod metrics.

**How CloudWatch and Prometheus/Grafana coexist**

* Use CloudWatch for infra-level metrics and logs (EKS node metrics, ALB, ELB, RDS metrics).
* Use Prometheus for application-level metrics (Keycloak counters/histograms). Connect Grafana to both CloudWatch and Prometheus/AMP as data sources.

---

## 7. Detailed architecture (recommended production)

* ALB (public) — TLS/ACM -> TargetGroup -> EKS (Keycloak service)
* EKS cluster (private subnets) hosting:

  * Keycloak Deployment (3 replicas) with HorizontalPodAutoscaler
  * Fluent Bit DaemonSet to push logs
  * Prometheus + ServiceMonitor
  * IAM Role for ServiceAccount for Secrets Manager / S3
* Aurora PostgreSQL (Multi-AZ)
* ElastiCache Redis cluster for sessions
* Secrets Manager for DB credentials and Keycloak `credential` secrets
* S3 bucket for themes & backups
* Managed Grafana and AMP (optional)

Diagram (text):

```
Internet -> Route53 -> ALB (ACM TLS) -> EKS Service -> Keycloak Pods
                          |-> CloudFront (optional) for global CDN
EKS -> SecretsManager / S3
Keycloak -> Aurora PostgreSQL
Keycloak -> ElastiCache (Redis)
Metrics: Prometheus <-> Keycloak exporter; Grafana reads Prometheus and CloudWatch
```

---

## 8. Keycloak configuration notes

* **Database**: configure JDBC URL to Aurora/Postgres, set connection pool (HikariCP) with proper max connections
* **Cluster mode**: ensure Use of external cache (Redis/Infinispan) to avoid sticky sessions
* **Session cookie settings**: `KEYCLOAK_FRONTEND_URL`, `PROXY_ADDRESS_FORWARDING` enabled when behind ALB
* **TLS**: offload TLS at ALB, but enable HTTPS between ALB and pods if compliance requires
* **Realm export/import**: use `kcadm.sh` or admin REST endpoints. Keep an automated export schedule to S3
* **Admin account**: create via env var init or use `keycloak import` scripts; don’t hard-code credentials. Store admin password in Secrets Manager

---

## 9. Observability specifics — metrics to capture & dashboards

**Metrics to track**

* Requests per second, latency (95th/99th)
* Authentication success / failure counts
* Active sessions & login rate
* Token issuance and refresh counts
* DB connection count and slow queries
* JVM metrics: heap usage, GC pause times (if using JVM distribution)

**Dashboards**

* Keycloak Overview: auth rates, errors, active sessions
* JVM and Pod: heap, GC, CPU, memory, restarts
* DB: connection usage, replication lag, CPU, storage
* ALB: request count, 5xx rates, latencies

**Alerting**

* High failed login rate (could indicate brute-force) — alert and rate limit
* DB connections > 80% of pool
* Pod restarts > X in Y min
* High latency for token issuance
* Low free memory on nodes

---

## 10. Security hardening & best practices

* Use **TLS** end-to-end if required for compliance
* Enforce strong password policies in Keycloak
* Enable **Brute force detection** and account lockout
* Use **Client certificates** or mutual TLS for sensitive admin APIs
* Rotate secrets (use Secrets Manager with rotation if needed)
* Restrict admin console access by IP or separate admin ALB
* Enable audit logging — stream to CloudWatch Logs and S3 for retention

---

## 11. Step-by-step deployment (EKS + Helm) — concise commands

> Assumes you have: `kubectl`, `helm`, `aws` CLI configured with access to the target AWS account and an EKS cluster ready.

1. **Prepare AWS infra** (VPC, EKS, Aurora, ElastiCache, S3, ACM cert)

   * Create Aurora PostgreSQL (Multi-AZ)
   * Create ElastiCache Redis cluster
   * Create S3 bucket for Keycloak backups and themes
   * Create ACM certificate for `auth.example.com`

2. **Create Secrets in AWS**

   * Store DB creds in `aws secretsmanager` or SSM Parameter Store

3. **Install Keycloak via Helm (example)**

```bash
# add chart repo
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update

# create namespace
kubectl create ns keycloak

# create k8s secret for DB credentials (or use secrets manager/IRSA)
kubectl create secret generic keycloak-db-secret -n keycloak \
  --from-literal=DB_USER=keycloak --from-literal=DB_PASSWORD='SuperSecret'

# sample install (values should point to Aurora endpoint and redis)
helm install keycloak codecentric/keycloak -n keycloak -f keycloak-values.yaml
```

`keycloak-values.yaml` should configure:

* external database host/user/password
* env: `PROXY_ADDRESS_FORWARDING=true`, `KC_DB=postgres`
* readiness/liveness and resources
* service type: ClusterIP behind an Ingress/ALB

4. **Ingress / ALB**

* Use AWS Load Balancer Controller to create ALB from Kubernetes Ingress
* Annotate Ingress for ACM cert and target type

5. **Monitoring**

* Deploy Prometheus stack and Keycloak exporter; configure ServiceMonitor
* Configure Fluent Bit to push logs to CloudWatch Logs
* Connect Grafana to Prometheus and CloudWatch

6. **Post-deploy tasks**

* Import realms and test login flows
* Configure clients and OIDC settings for apps
* Run load test and tune DB pool / JVM

````

---

## 12. Alternative: ECS / Fargate quick notes
- Run Keycloak as a Docker container in a Service with desired count 3
- Use ALB + target group
- Task role for Secrets Manager and S3 access
- Use CloudWatch Logs driver for container logs
- Use Fargate autoscaling and ECS service discovery for internal access

---

## 13. Example Keycloak Helm `values.yaml` (minimal, adapt to your infra)
```yaml
replicaCount: 3
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 1
    memory: 2Gi
keycloak:
  username: admin
  password: CHANGE_ME
postgresql:
  enabled: false # using external Aurora
externalDatabase:
  host: your-aurora-cluster.cluster-xxxxxx.us-east-1.rds.amazonaws.com
  name: keycloak
  user: keycloak
  password: CHANGE_ME
proxyAddressForwarding: true
service:
  type: ClusterIP
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
  hosts:
    - host: auth.example.com
      paths: ['/']
redis:
  enabled: false # using ElastiCache
````

---

## 14. Operational runbook (upgrade/backout/restore)

* **Upgrade Keycloak**: perform rolling upgrade with 3 replicas. Ensure readiness checks pass and DB is compatible with schema changes. Backup DB and export realms to S3 before upgrade.
* **Restore**: restore Aurora snapshot, create new Keycloak cluster pointing at restored DB, import realm exports if needed.
* **Scaling**: increase Keycloak replicas and node group size. Monitor DB connection limits when scaling pods.

---

## 15. Costs (brief)

Major cost drivers:

* EKS node hours or ECS Fargate task hours
* Aurora RDS instance sizing and IO
* ElastiCache node cost
* ALB and data transfer
* S3 storage and request counts
* Managed Grafana / AMP (if used)

---

## 16. Checklist before go-live

* [ ] TLS (ACM) configured and certificate validated
* [ ] Route53 DNS entry created
* [ ] Secrets stored in Secrets Manager and accessed via IRSA
* [ ] Automated backups for DB and Keycloak realm exports
* [ ] Logging -> CloudWatch and monitoring -> Prometheus/Grafana configured
* [ ] Alerts created for key metrics
* [ ] Load testing completed
* [ ] Disaster recovery documented and tested

---

## 17. Extras / things you may have missed

* **Rate limiting / brute-force protection**: configure Keycloak or add WAF in front of ALB to mitigate abuse
* **SAML / OIDC connector dependencies**: if Keycloak acts as identity broker, ensure upstream IdPs are reachable and their certs are managed
* **SSO integrations**: test SSO flows for all apps across environments
* **Session stickiness**: prefer stateless with external cache; otherwise configure sticky sessions carefully
* **Health endpoints**: expose readiness and liveness endpoints for ALB health checks

---

## 18. Final recommendations

* Use **EKS + Helm** for flexibility and best observability.
* Use **Aurora PostgreSQL (Multi-AZ)** and **ElastiCache Redis** for reliable DB and session store.
* Use **CloudWatch for infra logs/metrics** and **Prometheus + Grafana** for application metrics & dashboards.
* Automate backups and secrets handling via Secrets Manager + IAM roles.

---

### Want this converted into:

* Terraform/EKS + RDS + ElastiCache code skeleton?
* A ready `values.yaml` and Helm install script with IRSA config?
* Step-by-step runbook with exact CLI commands and IAM policies?

Tell me which of the above you want next and I'll generate it (Terraform/CFN/Helm manifests/CI job) — I can produce the concrete files and commands next.
