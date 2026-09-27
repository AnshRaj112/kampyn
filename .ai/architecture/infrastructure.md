# KAMPYN Infrastructure Architecture

## 1. Purpose

This document defines the infrastructure architecture for KAMPYN.

The infrastructure must support:

- Production SaaS deployments.
- University self-hosted deployments.
- Horizontal scaling.
- Secure networking.
- Reliable deployments.
- Observability.
- Disaster recovery.
- Background processing.
- Database infrastructure.
- Search infrastructure.
- Object storage.
- Event infrastructure.
- Environment isolation.

Infrastructure must remain replaceable where practical.

Application code should depend on defined infrastructure interfaces rather than provider-specific implementation details.

---

# 2. Infrastructure Principles

KAMPYN infrastructure follows these principles:

1. Infrastructure must support application architecture, not dictate it.
2. Production infrastructure must be reproducible.
3. Configuration must be externalized.
4. Secrets must never be committed to source control.
5. Services must be independently observable.
6. Failures must be isolated where practical.
7. Resources must have explicit limits.
8. Deployments must be repeatable and reversible.
9. Stateful systems require backup and recovery strategies.
10. Self-hosted deployments must not depend on EXSOLVIA's infrastructure.

---

# 3. High-Level Infrastructure

The conceptual production architecture is:

```text id="q7m3x8"
                         Internet
                            │
                            ↓
                    DNS / CDN / WAF
                            │
                            ↓
                    Load Balancer / Edge
                            │
              ┌─────────────┴─────────────┐
              ↓                           ↓
       KAMPYN Frontend              KAMPYN API
         Next.js                       Go
              │                           │
              │                    ┌──────┼──────┐
              │                    ↓      ↓      ↓
              │                 Worker  Cache  Search
              │                    │      │      │
              │                    └──┬───┴──┬───┘
              │                       ↓      ↓
              │                  PostgreSQL MongoDB
              │                       │
              │                 Object Storage
              │
              └──────────────→ External Providers
```

The exact deployment topology may differ between managed cloud and self-hosted environments.

---

# 4. Cloud Platform

KAMPYN may use Google Cloud Platform as a primary production infrastructure provider.

The architecture should avoid unnecessary coupling to GCP-specific application behavior.

Provider-specific infrastructure should remain isolated behind:

- Deployment configuration.
- Infrastructure-as-code.
- Infrastructure adapters.
- Platform-specific modules.

Application business logic must not depend directly on GCP APIs unless the feature explicitly requires them.

---

# 5. Compute

KAMPYN backend workloads may run on containerized compute infrastructure.

Possible deployment models include:

```text id="m4n8q2"
Managed Containers
       or
Kubernetes
       or
Institution-managed Kubernetes
```

The application must remain container-compatible regardless of the selected orchestration environment.

---

# 6. Docker

Every deployable backend service should have a reproducible container image.

Container images should:

- Use minimal base images where practical.
- Pin important dependencies.
- Run as a non-root user where possible.
- Avoid unnecessary packages.
- Contain only required runtime artifacts.
- Expose the expected application port.
- Handle graceful termination.
- Provide health endpoints where appropriate.

Do not use development tooling in production images unless required.

---

# 7. Multi-Stage Builds

Production Docker images should use multi-stage builds where appropriate.

Conceptually:

```text id="p8v2m5"
Source
  ↓
Build Image
  ↓
Compile / Bundle
  ↓
Runtime Image
  ↓
Production
```

Build dependencies should not unnecessarily increase the final runtime image.

---

# 8. Container Configuration

Containers must receive configuration through the deployment environment.

Do not hardcode:

- Database URLs.
- Redis endpoints.
- API keys.
- Credentials.
- Production domains.
- Environment-specific settings.

Configuration should be injected through the platform's configuration mechanism.

---

# 9. Container Lifecycle

Services must correctly handle:

- Startup.
- Readiness.
- Shutdown.
- SIGTERM.
- Connection cleanup.
- Worker termination.
- In-flight request handling.

A container should not immediately terminate while valid work is still being processed.

---

# 10. Health Checks

Services should expose appropriate health endpoints.

Distinguish between:

```text id="f6m2q8"
Liveness
    ↓
Is the process functioning?

Readiness
    ↓
Can the instance receive traffic?
```

Readiness may depend on required infrastructure connectivity.

Liveness should not fail merely because an optional dependency is temporarily unavailable.

---

# 11. Graceful Shutdown

During deployment or scaling down:

```text id="k3n7p2"
Receive termination signal
        ↓
Stop accepting new work
        ↓
Finish or cancel active requests
        ↓
Stop background workers
        ↓
Close connections
        ↓
Exit
```

Graceful shutdown must respect platform termination deadlines.

---

# 12. Kubernetes

When Kubernetes is used, workloads should be represented through explicit resources.

Typical components include:

```text id="x8q4m1"
Deployment
Service
Ingress / Gateway
ConfigMap
Secret reference
HorizontalPodAutoscaler
Job
CronJob
PodDisruptionBudget
```

Only introduce resources that the workload actually requires.

---

# 13. Kubernetes Deployments

Stateless application services should generally use Kubernetes Deployments.

Deployments should define:

- Replica count.
- Resource requests.
- Resource limits.
- Health probes.
- Rolling update behavior.
- Pod security configuration.

Avoid running critical application services as manually managed individual Pods.

---

# 14. Resource Requests and Limits

Every production workload should have resource expectations.

Define:

```text id="n5r8m2"
CPU request
Memory request
CPU limit
Memory limit
```

where appropriate.

Resource settings should be based on measurements rather than arbitrary values.

Unbounded workloads can destabilize the cluster.

---

# 15. Horizontal Scaling

Stateless services should support horizontal scaling.

Example:

```text id="q7m3n8"
                Load Balancer
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
       API Pod    API Pod    API Pod
```

Instances must not depend on local memory for authoritative shared state.

Shared state should reside in appropriate infrastructure such as:

- PostgreSQL.
- MongoDB.
- Redis.
- Object storage.
- Event infrastructure.

---

# 16. Horizontal Pod Autoscaling

Autoscaling may use:

- CPU.
- Memory.
- Request rate.
- Queue depth.
- Custom application metrics.

Autoscaling must account for downstream capacity.

Scaling the API from:

```text id="w4n8q2"
5 instances → 50 instances
```

does not help if PostgreSQL can only safely handle the original connection volume.

---

# 17. Background Workers

Asynchronous workloads should run independently from latency-sensitive API processes where practical.

Examples:

```text id="c8m2p5"
Event Consumers
Search Indexing
Notifications
Report Generation
File Processing
Data Reconciliation
Scheduled Jobs
```

Architecture:

```text id="r6n3q8"
API
 ↓
Queue / Event Infrastructure
 ↓
Worker Pool
 ↓
Processing
```

Workers should have bounded concurrency.

---

# 18. Worker Scaling

Workers should scale according to actual workload.

Possible scaling signals:

- Queue depth.
- Event lag.
- Processing latency.
- CPU.
- Memory.
- External API limits.

Do not scale workers without considering the capacity of downstream systems.

---

# 19. Cron Jobs

Scheduled tasks should use an explicit scheduler.

Kubernetes `CronJob` or an equivalent platform service may be used.

Scheduled tasks must be:

- Idempotent.
- Observable.
- Retry-safe.
- Bounded.
- Safe if accidentally triggered more than once.

Never assume a scheduler guarantees exactly one execution.

---

# 20. Networking

Network architecture should separate:

```text id="v2m8q4"
Public Traffic
     ↓
Edge / Load Balancer
     ↓
Application Network
     ↓
Private Infrastructure
```

Databases and internal infrastructure should not be publicly exposed unless there is an explicit architectural requirement.

---

# 21. Private Networking

Prefer private networking for:

- PostgreSQL.
- MongoDB.
- Redis.
- Event brokers.
- Internal services.
- Administrative infrastructure.

Public exposure should be minimized.

---

# 22. Service-to-Service Communication

Internal services should communicate through defined protocols.

Possible mechanisms:

- HTTP/gRPC.
- Event infrastructure.
- Queue systems.

Communication should include:

- Authentication where required.
- Timeouts.
- Retries where safe.
- Correlation IDs.
- Observability.

Do not assume internal network traffic is inherently trusted.

---

# 23. TLS

TLS should protect external traffic.

Typical flow:

```text id="q5n2m8"
Client
  ↓ HTTPS
Edge
  ↓
Application
```

Internal TLS requirements depend on the deployment environment and threat model.

Self-hosted deployments must provide a documented secure configuration.

---

# 24. DNS

DNS should provide stable entry points for:

- Marketing website.
- Application.
- API.
- Institution-specific deployments.
- Administrative interfaces where applicable.

Example conceptual structure:

```text id="m7p3q8"
kampyn.com
www.kampyn.com
app.kampyn.com
api.kampyn.com

tenant.kampyn.com
```

Actual domain structure must be defined by deployment requirements.

---

# 25. Tenant Routing

If KAMPYN uses institution-specific domains or subdomains, tenant resolution must be explicit.

Possible inputs:

```text id="x4n8q2"
Hostname
Authenticated tenant membership
Explicit tenant context
```

The backend must validate tenant access.

A hostname must never be treated as sufficient authorization.

---

# 26. CDN and Edge

A CDN may serve:

- Static assets.
- Public images.
- Marketing content.
- Public files.

Dynamic authenticated traffic should be routed according to the application security model.

CDN caching must preserve tenant and authorization isolation.

---

# 27. Web Application Firewall

A WAF may protect public endpoints against common web attacks.

Potential protections include:

- Malicious request filtering.
- Rate limiting.
- Bot controls.
- IP restrictions.
- Request size limits.

WAF rules are defense in depth.

Application-level validation remains mandatory.

---

# 28. Rate Limiting

Rate limiting may exist at multiple layers:

```text id="p8m3q7"
Edge
 ↓
API
 ↓
Domain-specific operations
```

Examples of particularly sensitive endpoints:

- Login.
- Password reset.
- OTP.
- Search.
- File uploads.
- Payment operations.
- Community posting.

Limits must not interfere with legitimate institutional workloads.

---

# 29. Secrets Management

Secrets must be stored using a dedicated secrets mechanism.

Examples:

```text id="n2v7m4"
Cloud Secret Manager
Kubernetes Secret integration
Institution-controlled secret store
```

Never commit:

```text id="q5m8p2"
.env
.env.production
private keys
API secrets
database passwords
```

to source control.

---

# 30. Secret Rotation

Secrets must be rotatable without requiring source-code changes.

Examples:

- Database credentials.
- API keys.
- Signing keys.
- OAuth credentials.
- Webhook secrets.

Rotation procedures should be documented.

---

# 31. Environment Separation

KAMPYN environments should be isolated.

Typical environments:

```text id="c7n2m8"
Local
Development
Testing
Staging
Production
```

Each environment should have independent:

- Credentials.
- Databases.
- Cache.
- Event infrastructure where appropriate.
- Storage.
- External service configuration.

Production data should not be casually copied into development environments.

---

# 32. Environment Configuration

Configuration should be separated from code.

Conceptually:

```text id="v4m8q2"
Application Image
       +
Environment Configuration
       +
Secrets
       ↓
Running Service
```

The same immutable application image should ideally be deployable across environments with different configuration.

---

# 33. Infrastructure as Code

Production infrastructure should be reproducible through infrastructure-as-code where practical.

Infrastructure-as-code should manage:

- Networks.
- Compute.
- Databases where supported.
- Storage.
- IAM.
- Kubernetes infrastructure.
- DNS.
- Monitoring.
- Secrets references.

Manual infrastructure changes should be minimized and documented.

---

# 34. Infrastructure State

Infrastructure state must be stored securely and centrally.

State must not be:

- Committed accidentally.
- Stored in local developer machines as the only copy.
- Exposed publicly.

Infrastructure state may contain sensitive resource information.

---

# 35. IAM

Infrastructure access should follow least privilege.

Separate roles for:

```text id="m3q8n2"
Developers
CI/CD
Runtime services
Infrastructure administrators
Database administrators
Security administrators
```

Application workloads should not receive broad cloud administrator permissions.

---

# 36. Service Accounts

Each workload should use an appropriate service identity.

Avoid sharing one powerful service account across unrelated services.

Prefer:

```text id="r8n2m5"
API Service Account
Worker Service Account
Deployment Service Account
Monitoring Service Account
```

with only the required permissions.

---

# 37. CI/CD

CI/CD should automate:

```text id="k4m7q2"
Commit
 ↓
Validation
 ↓
Tests
 ↓
Build
 ↓
Security Checks
 ↓
Container Image
 ↓
Deployment
 ↓
Health Verification
```

Production deployment should not depend on a developer manually rebuilding artifacts.

---

# 38. Container Registry

Production images should be stored in a controlled container registry.

Images should have:

- Immutable tags or digests.
- Vulnerability scanning.
- Access control.
- Retention policy.

Production deployments should prefer immutable image references.

---

# 39. Image Versioning

Avoid relying exclusively on:

```text id="p2n8m4"
latest
```

for production deployments.

Prefer explicit versions or immutable digests:

```text id="v7q3m8"
kampyn-api:1.4.2
```

or:

```text id="n4m2p7"
sha256:...
```

This makes deployments reproducible.

---

# 40. Deployment Strategy

Deployment strategy should minimize service disruption.

Possible strategies include:

```text id="c8m4q2"
Rolling Deployment
Blue/Green
Canary
```

Choose based on:

- Service criticality.
- Infrastructure complexity.
- Database compatibility.
- Traffic volume.
- Rollback requirements.

Do not introduce sophisticated deployment strategies without operational justification.

---

# 41. Database Deployments

Application deployment and database migration must be coordinated.

Prefer:

```text id="x7n2m5"
Backward-compatible migration
        ↓
Application deployment
        ↓
Data migration
        ↓
Cleanup migration
```

Avoid migrations that make rollback impossible unless the deployment process explicitly accounts for that.

---

# 42. Rollbacks

Every production deployment should have a rollback strategy.

Possible rollback mechanisms:

```text id="m8q4n2"
Previous container image
Previous deployment version
Feature flag
Database-compatible rollback
```

Database schema changes may make application rollback unsafe.

Therefore migration compatibility must be considered before deployment.

---

# 43. Zero-Downtime Deployment

Zero-downtime deployment requires compatibility between:

```text id="p3v8m2"
Old Application
New Application
Current Database
```

Services must remain available while instances are replaced.

Health checks must prevent unhealthy instances from receiving traffic.

---

# 44. Observability Architecture

Infrastructure observability should include:

```text id="q5m2n8"
Logs
Metrics
Traces
Alerts
Health Checks
```

Conceptually:

```text id="c8r4m7"
Application
    │
    ├── Logs
    ├── Metrics
    └── Traces
          │
          ↓
   Observability Platform
          │
          ↓
        Alerts
```

---

# 45. Logging

Logs should be:

- Structured.
- Machine-readable.
- Correlated.
- Environment-aware.
- Free of unnecessary sensitive data.

Include useful fields such as:

```text id="v7m3q2"
timestamp
service
environment
request_id
correlation_id
tenant_id where appropriate
severity
message
error_code
```

Never log:

- Passwords.
- Authentication tokens.
- Private keys.
- Secrets.
- Sensitive payloads unnecessarily.

---

# 46. Metrics

Infrastructure and application metrics should cover:

- Request rate.
- Request latency.
- Error rate.
- CPU.
- Memory.
- Database connections.
- Queue depth.
- Event lag.
- Cache hit rate.
- Worker throughput.
- External API failures.

Metrics should be actionable rather than excessive.

---

# 47. Distributed Tracing

Distributed tracing should connect:

```text id="n8m4q2"
HTTP Request
    ↓
API
    ↓
Database
    ↓
Event
    ↓
Worker
    ↓
External Service
```

Use correlation and trace identifiers consistently.

Tracing should avoid storing sensitive payloads.

---

# 48. Alerting

Alerts should represent conditions requiring attention.

Examples:

```text id="r2m7q8"
High API error rate
Database saturation
Queue backlog
High event lag
Disk exhaustion
Repeated deployment failure
Certificate expiry
Backup failure
```

Avoid alerting on every transient error.

Alerts must have an operational response.

---

# 49. Monitoring Database Infrastructure

Monitor:

- Storage.
- CPU.
- Memory.
- Connections.
- Query latency.
- Replication.
- Backup status.
- Lock contention.

Database alerts should account for both immediate symptoms and long-term capacity.

---

# 50. Storage

Infrastructure storage should be classified by purpose.

```text id="m5q8n2"
Database Storage
Object Storage
Ephemeral Container Storage
Logs
Backups
```

Do not rely on container-local storage for durable application data.

---

# 51. Object Storage Architecture

Object storage should handle large files.

Typical flow:

```text id="x7n3p8"
Client
  ↓
KAMPYN API
  ↓
Authorization
  ↓
Upload URL / Upload Service
  ↓
Object Storage
```

Large files should not unnecessarily pass through application servers.

Where direct uploads are used, authorization and upload constraints must be enforced.

---

# 52. Object Storage Security

Object storage should use:

- Private buckets by default.
- Controlled access.
- Signed URLs where appropriate.
- Content-type validation.
- Size limits.
- Lifecycle policies.
- Encryption.

Never expose a private bucket publicly merely to simplify frontend access.

---

# 53. Backups

Persistent systems require backups.

Backup targets may include:

```text id="c4m8q2"
PostgreSQL
MongoDB
Object Storage
Infrastructure configuration
Critical event data
```

Redis may not require traditional backups when used purely as a disposable cache.

The recovery strategy determines which systems require durable backup.

---

# 54. Disaster Recovery

Disaster recovery should define:

- Recovery Point Objective.
- Recovery Time Objective.
- Backup location.
- Restoration procedure.
- Infrastructure recreation procedure.
- DNS recovery.
- Credential recovery.
- Data validation.

Recovery must be tested periodically.

---

# 55. Regional Strategy

The initial deployment may operate in a single region when that provides sufficient reliability.

Multi-region infrastructure should only be introduced when justified by:

- Availability requirements.
- Disaster recovery requirements.
- User geography.
- Regulatory requirements.
- Business continuity.

Multi-region architecture significantly increases operational complexity.

---

# 56. Availability Zones

Where supported and appropriate, production workloads should avoid unnecessary single-instance dependencies.

Stateless services should be distributed across available failure domains when the workload justifies it.

Stateful systems require their own high-availability strategy.

---

# 57. Infrastructure Failure Isolation

Failures should remain bounded.

Examples:

```text id="m8q3n7"
Search Failure
    ↓
Ordering still works

Notification Failure
    ↓
Order creation still works

Cache Failure
    ↓
Authoritative database remains available
```

The architecture should avoid making optional dependencies mandatory for unrelated operations.

---

# 58. Dependency Timeouts

Every network dependency should have an appropriate timeout.

Examples:

- Database.
- Redis.
- OpenSearch.
- Event broker.
- Payment provider.
- Email provider.
- External institutional services.

Never allow network calls to wait indefinitely.

---

# 59. Retries

Retries must be:

- Bounded.
- Backoff-based.
- Idempotent where required.
- Limited by dependency capacity.

Do not retry every error.

Avoid retry storms:

```text id="x4m8q2"
Service fails
 ↓
100 requests retry
 ↓
100 requests retry again
 ↓
Dependency becomes more overloaded
```

---

# 60. Circuit Breaking

Circuit breakers may be appropriate for unstable external dependencies.

Example:

```text id="q7n3m8"
External Service
      ↓
Repeated Failure
      ↓
Circuit Opens
      ↓
Fail Fast
      ↓
Periodic Recovery Attempt
```

Use circuit breakers where they provide meaningful protection.

Do not add them to every internal call automatically.

---

# 61. Queue Backpressure

Asynchronous infrastructure must protect downstream dependencies.

For example:

```text id="v5m8q2"
10,000 events
      ↓
Queue
      ↓
Bounded workers
      ↓
Database
```

not:

```text id="n2q7m4"
10,000 events
      ↓
10,000 goroutines
      ↓
Database exhaustion
```

---

# 62. Infrastructure Security

Infrastructure must implement defense in depth.

Consider:

- Network isolation.
- IAM.
- TLS.
- Secret management.
- Container security.
- Image scanning.
- Dependency scanning.
- Firewall rules.
- WAF.
- Runtime restrictions.
- Audit logging.

Security controls should be layered rather than relying on one boundary.

---

# 63. Kubernetes Security

Where Kubernetes is used:

- Run workloads as non-root where possible.
- Use restrictive security contexts.
- Limit capabilities.
- Restrict network access.
- Use RBAC.
- Protect service accounts.
- Avoid privileged containers.
- Restrict host access.
- Use resource limits.
- Scan images.

Do not grant cluster-wide permissions to application workloads unnecessarily.

---

# 64. Network Policies

Network policies may restrict which workloads can communicate.

Example:

```text id="p8m4q2"
Frontend
   ↓
API

API
   ↓
Database
   ↓
Redis
   ↓
OpenSearch
```

The frontend should not automatically have direct access to database infrastructure.

---

# 65. Infrastructure Documentation

Every production infrastructure component should have documentation covering:

- Purpose.
- Owner.
- Dependencies.
- Deployment.
- Configuration.
- Health checks.
- Scaling.
- Failure behavior.
- Backup.
- Recovery.
- Security.
- Troubleshooting.

Infrastructure knowledge must not exist only in one engineer's memory.

---

# 66. Runbooks

Critical operational workflows should have runbooks.

Examples:

```text id="m7q3n8"
Database outage
Redis outage
OpenSearch failure
Event backlog
Failed deployment
Certificate expiry
Backup restoration
Credential rotation
Kubernetes node failure
```

Runbooks should contain concrete recovery steps.

---

# 67. Self-Hosted Architecture

KAMPYN must support university-controlled deployments.

A conceptual self-hosted topology:

```text id="q4n8m2"
University Network
        │
        ↓
   Reverse Proxy
        │
 ┌──────┴───────┐
 ↓              ↓
Frontend       API
                  │
        ┌─────────┼──────────┐
        ↓         ↓          ↓
   PostgreSQL   MongoDB    Redis
        │
   Object Storage
        │
   Event/Search Infrastructure
```

The exact topology may vary according to institutional infrastructure.

---

# 68. Self-Hosted Configuration

Self-hosted deployments should be configurable without modifying application source.

Configuration should cover:

- Domains.
- Database endpoints.
- Redis.
- OpenSearch.
- Event broker.
- Object storage.
- Authentication providers.
- Email/SMS providers.
- Payment providers where applicable.
- Feature configuration.

---

# 69. Self-Hosted Upgrades

Self-hosted institutions should have a documented upgrade process:

```text id="n8m4q2"
Backup
  ↓
Compatibility Check
  ↓
Database Migration
  ↓
Application Upgrade
  ↓
Health Verification
  ↓
Post-Upgrade Validation
```

Version compatibility must be explicit.

---

# 70. Infrastructure Cost

Infrastructure decisions should consider:

- Compute.
- Database.
- Storage.
- Network traffic.
- Logs.
- Search.
- Event infrastructure.
- Backups.
- Monitoring.

Do not introduce infrastructure solely because it is technically interesting.

Every infrastructure component creates:

```text id="x3q8m2"
Operational Cost
+
Financial Cost
+
Security Surface
+
Maintenance Cost
```

---

# 71. Infrastructure Capacity Planning

Capacity planning should consider:

- Current traffic.
- Peak traffic.
- Enrollment growth.
- Order volume.
- Search volume.
- Event volume.
- Database growth.
- File storage growth.
- Institutional tenant count.

Plan for meaningful growth rather than arbitrary theoretical scale.

---

# 72. Load Testing

Production infrastructure should be load-tested before major scale increases.

Test:

- API throughput.
- Database behavior.
- Cache behavior.
- Search.
- Event processing.
- Worker scaling.
- File uploads.
- Peak concurrent users.

Load tests should identify bottlenecks rather than simply produce a single throughput number.

---

# 73. Infrastructure Change Management

Infrastructure changes must be reviewed like application changes.

Consider:

- Security.
- Availability.
- Cost.
- Rollback.
- Data compatibility.
- Scaling.
- Monitoring.
- Self-hosted implications.

Avoid undocumented manual changes to production infrastructure.

---

# 74. Infrastructure Change Checklist

Before completing an infrastructure-related change:

- [ ] Infrastructure ownership is clear.
- [ ] Deployment model is defined.
- [ ] Configuration is externalized.
- [ ] Secrets are protected.
- [ ] Network exposure is minimized.
- [ ] Resource limits are defined.
- [ ] Health checks are configured.
- [ ] Graceful shutdown is supported.
- [ ] Horizontal scaling behavior is understood.
- [ ] Database capacity is considered.
- [ ] Cache capacity is considered.
- [ ] Event/worker capacity is considered.
- [ ] Timeouts are configured.
- [ ] Retry behavior is bounded.
- [ ] Observability is available.
- [ ] Alerts are meaningful.
- [ ] Backup/recovery implications are understood.
- [ ] Rollback is possible.
- [ ] Security implications are reviewed.
- [ ] Self-hosted deployment implications are considered.
- [ ] Documentation and runbooks are updated.

---

# 75. Final Infrastructure Principle

KAMPYN infrastructure should make the application:

```text id="v8m3q2"
Deployable
Observable
Scalable
Recoverable
Secure
Portable
```

The preferred operational architecture is:

```text id="c7q2m8"
                         Internet
                            │
                       DNS / Edge
                            │
                     Load Balancer
                            │
             ┌──────────────┴──────────────┐
             ↓                             ↓
        Next.js Frontend               Go API
                                           │
                     ┌─────────────────────┼──────────────────┐
                     ↓                     ↓                  ↓
                  Workers                Redis            OpenSearch
                     │                     │                  │
                     └──────────────┬──────┴──────────────────┘
                                    ↓
                         Authoritative Databases
                           ┌────────┴────────┐
                           ↓                 ↓
                      PostgreSQL          MongoDB
                           │
                           ↓
                     Object Storage
```

Infrastructure should remain:

```text id="m4n8q2"
Reproducible
    ↓
Secure
    ↓
Observable
    ↓
Fault-tolerant
    ↓
Scalable
    ↓
Operationally simple
```

The infrastructure exists to provide a reliable execution environment for KAMPYN's application architecture.

It should not become a second application architecture.

Every infrastructure component must have a clear purpose, owner, failure model, security boundary, scaling strategy, and recovery strategy.