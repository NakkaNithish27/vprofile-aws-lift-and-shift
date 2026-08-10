# Limitations & Future Work

[← Back to README](../README.md) · [← Architecture](architecture.md) · [← Implementation](implementation.md)

---

## 1. Purpose

This document defines the boundaries of the AWS lift-and-shift project and identifies logical directions for future evolution.

The purpose is to make an explicit distinction between:

* what this project actually demonstrates
* what it does not demonstrate
* what could be implemented in a future iteration

The project should therefore be understood as a **lift-and-shift / IaaS deployment**, not as a complete cloud-native or production engineering platform. The source material explicitly describes the final architecture as an IaaS implementation using EC2 instances while preserving the original application architecture.

---

# 2. Current Project Scope

The project successfully demonstrates:

* AWS EC2 infrastructure
* multi-tier application deployment
* layered Security Groups
* Route 53 Private DNS
* S3-based application artifact transfer
* IAM-based EC2 access to AWS services
* Apache Tomcat deployment
* Application Load Balancer
* Target Groups
* HTTPS with ACM
* public DNS mapping
* AMI creation
* Launch Templates
* Auto Scaling Groups
* Target Tracking scaling
* ELB health checks
* application/backend connectivity validation

The final architecture applies Auto Scaling specifically to the **Tomcat application tier**. The database, Memcache, and RabbitMQ services remain manually managed EC2 instances.

---

# 3. Application Ownership Limitation

The vProfile application was used as the workload for the project.

The project does **not** establish ownership of:

* the application's Java source code
* application business logic
* application architecture
* application authentication implementation
* the underlying application functionality

The engineering contribution represented by this repository is the AWS infrastructure and deployment environment around the existing workload.

Therefore, the project should not be described as:

> "Developed the vProfile application."

The accurate boundary is:

> **Deployed and operated the existing vProfile workload on AWS using a lift-and-shift approach.**

---

# 4. Lift-and-Shift Boundary

The project intentionally preserves the application's existing architecture.

The fundamental transformation is:

```text
Local / Vagrant Environment
          │
          │ Lift & Shift
          ▼
AWS EC2 + AWS Services
```

The application architecture itself was not substantially redesigned.

The source material describes the migration as:

```text
Same Application
Same Application Architecture
Different Infrastructure
```

This means the project demonstrates **infrastructure migration**, rather than cloud-native application modernization.

---

# 5. Infrastructure as a Service, Not Infrastructure as Code

The AWS environment was provisioned through the practical execution workflow using AWS resources such as EC2, Security Groups, Route 53, S3, ALB, ACM, AMI, Launch Templates, and Auto Scaling Groups.

This project therefore demonstrates:

> **Infrastructure as a Service (IaaS)**

It does **not** demonstrate a complete Infrastructure-as-Code implementation.

There is no Terraform or CloudFormation implementation represented by this repository.

This distinction is important because the source material explicitly clarifies that the final architecture is IaaS rather than Infrastructure as Code.

---

# 6. Manual Build and Deployment

The application artifact follows this flow:

```text
Local Machine
      │
      ▼
    Maven
      │
      ▼
     WAR
      │
      ▼
      S3
      │
      ▼
 Tomcat EC2
```

The project therefore demonstrates the **manual version of a Build → Store → Deploy pattern**.

The source material notes that a mature implementation could replace the local build step with a CI/CD system while retaining the same underlying artifact flow.

### What is not demonstrated

This project does not implement:

* Jenkins
* GitHub Actions
* GitLab CI/CD
* automated build triggers
* automated testing pipelines
* automated deployment pipelines
* automated rollback

These are future capabilities rather than completed capabilities.

---

# 7. Application-Tier-Only Auto Scaling

Auto Scaling is applied specifically to the Tomcat application tier.

```text
                 ALB
                  │
                  ▼
        Application Tier
                  │
             Auto Scaling
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Tomcat    Tomcat    Tomcat
```

The backend services remain individually managed:

```text
MySQL
Memcache
RabbitMQ
```

They are not part of the Auto Scaling Group.

The reason is that the application tier is the component directly receiving user traffic and is therefore the natural candidate for horizontal scaling in this project. The backend services have different state and lifecycle characteristics.

---

# 8. Backend High Availability Is Not Demonstrated

The application tier has an Auto Scaling mechanism, but the backend services remain individually managed.

The architecture therefore does not establish full-stack high availability.

For example:

```text
Application Tier
      │
      ▼
   Auto Scaled
      │
      ├──────────────┐
      │              │
      ▼              ▼
  Database       RabbitMQ
      │              │
      └──────┬───────┘
             ▼
          Memcache
```

A failure of an individually managed backend instance is not automatically handled in the same way as failure of an ASG-managed application instance.

This is an important architectural limitation.

---

# 9. Session Management Limitation

The validation material identifies an important limitation when multiple application instances are placed behind the load balancer.

The application may depend on session behavior that is not externally managed.

When multiple instances are active, load-balancer stickiness may be required to keep a user's requests associated with the appropriate application instance.

Conceptually:

```text
Without shared session state:

User
 │
 ├── Request 1 ──► Instance A
 │
 └── Request 2 ──► Instance B
                       │
                       ▼
                 Session unavailable
```

Target-group stickiness provides a pragmatic solution:

```text
User
 │
 ├── Request 1 ──► Instance A
 │
 ├── Request 2 ──► Instance A
 │
 └── Request 3 ──► Instance A
```

However, stickiness creates a tradeoff because traffic may become unevenly distributed between instances.

The source material identifies **externalized session management** as the more cloud-native solution, using a shared session store such as Memcache or Redis so that any application instance can handle a request.

---

# 10. No Application Re-Architecture

The project does not modify the application to take advantage of cloud-native patterns.

The application remains fundamentally tied to the original multi-tier design.

The project therefore does not demonstrate:

* stateless application redesign
* containerization
* service decomposition
* managed database migration
* managed messaging migration
* cloud-native session management

These would require changes beyond the lift-and-shift scope.

---

# 11. No Production Data Migration

The project demonstrates migration of the **application and infrastructure environment**.

It does not establish a production database migration process.

The source material's final-state description explicitly distinguishes the infrastructure migration from migration of production data.

Therefore this project does not demonstrate:

* production database synchronization
* database replication for migration
* production cutover
* rollback of a production data migration
* migration of a live production dataset

---

# 12. No Production Observability Platform

The project validates the system through infrastructure and application checks, including:

* EC2/service validation
* DNS validation
* Target Group health
* application access
* backend connectivity
* Auto Scaling validation

It does not establish a complete observability platform.

A future implementation could introduce:

* centralized logs
* application metrics
* infrastructure dashboards
* alerting
* distributed tracing
* operational dashboards

These capabilities were not part of the demonstrated project scope.

---

# 13. No Automated Infrastructure Reproduction

Although the project uses reusable AWS mechanisms such as:

```text
AMI
  ↓
Launch Template
  ↓
Auto Scaling Group
```

the overall AWS environment is not represented as a fully declarative infrastructure definition.

The Auto Scaling workflow does make the **application tier** reproducible at the instance level, but the complete environment still depends on manually configured AWS resources.

The source material identifies the AMI → Launch Template → ASG chain as the mechanism used to reproduce application instances.

---

# 14. No Automated Testing Pipeline

The project validates the deployed application manually.

Validation includes application behavior and backend integration, such as:

```text
Login
  ↓
Database connectivity

Application behavior
  ↓
RabbitMQ connectivity

User retrieval
  ↓
Memcache behavior
```

The source material describes these application-level checks as part of the final validation.

The project does not establish an automated test suite or automated regression pipeline.

---

# 15. No Automated Deployment Rollback

The artifact deployment process is:

```text
Build
 ↓
S3
 ↓
Tomcat
```

There is no demonstrated automated rollback mechanism for a failed application release.

A future deployment pipeline could introduce:

* artifact versioning
* deployment validation
* automatic rollback
* release promotion
* deployment history

---

# 16. Future Work

The following improvements represent logical next steps for the project.

They are **not implemented capabilities**.

---

## 16.1 Infrastructure as Code

Rebuild the AWS environment using a declarative Infrastructure-as-Code tool such as Terraform.

The target architecture would move from:

```text
Manual AWS Configuration
          │
          ▼
     AWS Resources
```

to:

```text
Terraform Configuration
          │
          ▼
     AWS Resources
```

This would make the complete infrastructure reproducible, reviewable, and version-controlled.

---

## 16.2 CI/CD Pipeline

Replace the manual artifact flow:

```text
Local Machine
     ↓
Maven
     ↓
S3
     ↓
Tomcat
```

with an automated pipeline:

```text
Git Push
   ↓
CI Build
   ↓
Automated Tests
   ↓
Artifact
   ↓
Artifact Storage
   ↓
Deployment
   ↓
Validation
```

The source material explicitly identifies CI/CD as the natural evolution of the manual Build → Store → Deploy pattern.

---

## 16.3 Externalized Session Management

Replace dependence on load-balancer stickiness with shared session state.

Possible architecture:

```text
             ALB
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
     App A  App B  App C
       │      │      │
       └──────┼──────┘
              │
              ▼
       Shared Session Store
```

This would allow any application instance to handle a user's request without depending on session affinity.

The existing Memcache component could be investigated as part of such an evolution, although changing the application's session behavior would require application-level work outside the current project scope.

---

## 16.4 Backend High Availability

A future architecture could introduce high-availability mechanisms for the backend services.

The objective would be to remove the single-instance dependency currently present in the manually managed backend tier.

Potential future areas include:

```text
Database
   ↓
High-availability database architecture

RabbitMQ
   ↓
Highly available broker architecture

Memcache
   ↓
Distributed / managed caching architecture
```

The exact implementation would require additional architectural decisions beyond this project.

---

## 16.5 Managed AWS Services

The lift-and-shift architecture could gradually replace self-managed infrastructure with appropriate AWS-managed services.

The evolution could look like:

```text
EC2-based Lift & Shift
        ↓
Automated EC2 Infrastructure
        ↓
Managed AWS Services
        ↓
Cloud-Native Architecture
```

This would reduce the amount of infrastructure that must be manually maintained.

---

## 16.6 Observability

Introduce a dedicated observability layer covering:

```text
Metrics
   +
Logs
   +
Alerts
   +
Dashboards
```

The goal would be to move from:

> "The application works when manually checked."

toward:

> "The system continuously reports its operational health."

---

## 16.7 Automated Validation

The current validation is primarily manual.

A future iteration could automate checks for:

* EC2/service health
* DNS resolution
* Target Group health
* application availability
* backend connectivity
* Auto Scaling state

This would turn the validation process into a repeatable verification workflow.

---

## 16.8 Automated AMI / Image Creation

The current Auto Scaling implementation starts from an AMI created from a known-good application instance.

A future implementation could automate the image creation process:

```text
Application Version
        ↓
Image Build
        ↓
Validated AMI
        ↓
Launch Template
        ↓
Auto Scaling Group
```

This would reduce dependence on manually prepared application instances.

---

# 17. Potential Evolution Path

The project can evolve incrementally rather than being completely redesigned at once.

```text
Phase 1
AWS Lift & Shift
        │
        ▼
Phase 2
Infrastructure as Code
        │
        ▼
Phase 3
CI/CD
        │
        ▼
Phase 4
Automated Validation
        │
        ▼
Phase 5
Observability
        │
        ▼
Phase 6
Externalized Sessions
        │
        ▼
Phase 7
Backend High Availability
        │
        ▼
Phase 8
Managed / Cloud-Native Services
```

This progression preserves the engineering knowledge gained from the current project while adding automation and operational maturity incrementally.

---

# 18. What Should Not Be Claimed

Based on the current project scope, the following should **not** be presented as completed capabilities:

* Terraform
* Infrastructure as Code
* CI/CD
* Jenkins
* GitHub Actions
* Kubernetes
* Docker
* production-grade observability
* full-stack high availability
* production database migration
* cloud-native architecture
* application development
* automated deployment rollback
* externalized application session management

These belong either to future work or outside the demonstrated project scope.

---

# 19. Final Boundary

The project's engineering story is:

```text
Existing Multi-Tier Application
             │
             ▼
       AWS Lift & Shift
             │
             ├── EC2
             ├── Security Groups
             ├── Route 53
             ├── S3
             ├── IAM
             ├── ALB
             ├── ACM / HTTPS
             └── Auto Scaling
             │
             ▼
      Validated AWS Deployment
```

The project successfully demonstrates the infrastructure engineering required to move an existing workload to AWS and introduce application-tier elasticity.

Its main limitation is intentional:

> **The project changes the infrastructure around the application without fundamentally changing the application itself.**

That limitation also defines the natural next stage of the engineering journey:

```text
Lift & Shift
     ↓
Automate
     ↓
Observe
     ↓
Decouple
     ↓
Increase Resilience
     ↓
Modernize
```

The future capabilities described in this document should be treated as **proposed next iterations**, not as capabilities already implemented by this project.

---

[← Back to README](../README.md) · [← Architecture](architecture.md) · [← Implementation](implementation.md) · [Validation →](validation.md)
