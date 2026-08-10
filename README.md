# AWS Lift & Shift — vProfile Multi-Tier Application

AWS lift-and-shift deployment of an existing multi-tier Java application, using Amazon EC2, Route 53, S3, IAM, Application Load Balancer, ACM, and Auto Scaling.

> **Project focus:** AWS infrastructure, application deployment, service connectivity, secure ingress, application-tier scaling, and end-to-end validation.

---

## Overview

This project demonstrates the migration of an existing multi-tier application from a local virtual-machine environment to AWS using a **lift-and-shift** approach.

The application architecture was preserved rather than re-engineered. The primary engineering work was performed around the application by provisioning and configuring the AWS infrastructure required to run it.

The resulting environment uses:

* Amazon EC2 for the application and backend service tiers
* Security Groups for layered network access control
* Route 53 Private DNS for backend service discovery
* Amazon S3 for application artifact storage
* IAM for AWS resource access
* Apache Tomcat for the application tier
* Application Load Balancer for public application traffic
* AWS Certificate Manager for HTTPS
* Amazon Machine Image and Launch Template for application-instance replication
* Auto Scaling Group for application-tier lifecycle and scaling
* Target Group health checks for application-instance validation

The project was built and validated as a complete end-to-end deployment before the Auto Scaling layer was added.

---

## Application Ownership

### Important scope boundary

The **vProfile application was used as the workload for this project and was not developed as part of this work**.

The application source, application implementation, and course-provided provisioning artifacts originated from the supplied project material.

My work focused on the infrastructure and operational engineering required to deploy and run the existing application on AWS, including:

* AWS infrastructure provisioning
* network and security configuration
* backend service connectivity
* private DNS configuration
* application artifact deployment
* load balancing
* HTTPS configuration
* application-tier Auto Scaling
* health validation
* end-to-end system validation
* AWS resource cleanup

This repository therefore represents the **AWS engineering performed around the application**, rather than the development of the application itself.

---

## Architecture

![AWS vProfile Lift & Shift Architecture](architecture.png)

### High-level architecture

```text
                           Internet
                              │
                              │ HTTPS
                              ▼
                    ┌─────────────────────┐
                    │ Application Load    │
                    │ Balancer            │
                    │                     │
                    │ HTTP / HTTPS        │
                    └──────────┬──────────┘
                               │
                               │ HTTP :8080
                               ▼
                    ┌─────────────────────┐
                    │ Application Tier    │
                    │                     │
                    │ Tomcat              │
                    │ Auto Scaling Group  │
                    └──────────┬──────────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
                 ▼             ▼             ▼
          ┌───────────┐ ┌───────────┐ ┌───────────┐
          │  MySQL /  │ │ Memcache  │ │ RabbitMQ  │
          │  MariaDB  │ │           │ │           │
          └───────────┘ └───────────┘ └───────────┘
                 ▲             ▲             ▲
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                         Route 53 Private
                              DNS
```

The application tier is separated from the backend services through security-group boundaries and communicates with the backend services using private DNS names rather than directly depending on their private IP addresses.

---

## Architecture Components

| Component                 | Role                                                     |
| ------------------------- | -------------------------------------------------------- |
| Amazon EC2                | Hosts the application and backend services               |
| Application Load Balancer | Public entry point for application traffic               |
| Target Group              | Registers and health-checks Tomcat instances             |
| Auto Scaling Group        | Manages the application-tier instances                   |
| AMI                       | Provides the application-instance image used for scaling |
| Launch Template           | Defines how new application instances are launched       |
| Route 53 Private DNS      | Provides internal service discovery                      |
| Amazon S3                 | Stores the deployable application artifact               |
| IAM                       | Controls AWS access for artifact operations              |
| ACM                       | Provides the TLS certificate used by HTTPS               |
| Security Groups           | Enforce traffic boundaries between tiers                 |
| Tomcat                    | Runs the supplied Java web application                   |
| MySQL / MariaDB           | Database service                                         |
| Memcache                  | Caching service                                          |
| RabbitMQ                  | Message-broker service                                   |

---

# Engineering Objective

The original application stack consisted of multiple services running on local virtual machines.

The objective of the project was to move that workload to AWS while keeping the application's fundamental architecture intact.

The migration therefore followed a lift-and-shift model:

```text
Existing Application
        │
        │ minimal application change
        ▼
AWS Infrastructure
        │
        ├── EC2
        ├── Route 53
        ├── S3
        ├── ALB
        ├── ACM
        └── Auto Scaling
```

The focus was on changing the **hosting and operational environment**, rather than redesigning the application itself.

---

# My Engineering Contribution

My work on the project covered the following areas.

## 1. AWS Infrastructure

Provisioned the EC2-based infrastructure required for the application stack.

This included separate instances for:

* application service
* database
* caching
* message broker

The instances were configured with project-specific names and tags to make the environment easier to identify and manage.

---

## 2. Network Security

Implemented layered Security Groups corresponding to the major application tiers.

The intended traffic path was:

```text
Internet
   │
   │ 443
   ▼
Load Balancer Security Group
   │
   │ 8080
   ▼
Application Security Group
   │
   │ backend service ports
   ▼
Backend Security Group
```

The application tier was not intended to receive normal application traffic directly from the public internet.

Instead, application traffic was routed through the Application Load Balancer.

Backend services were restricted to traffic originating from the application tier.

This establishes a layered security boundary between:

* public ingress
* application processing
* backend services

---

## 3. Private Service Discovery

Configured Route 53 Private DNS records for the backend services.

The application uses service names such as:

```text
db01.vprofile.in
mc01.vprofile.in
rmq01.vprofile.in
```

These names resolve inside the VPC to the corresponding backend instance private IP addresses.

The resulting dependency flow is:

```text
Tomcat
   │
   ├── db01.vprofile.in
   │          │
   │          ▼
   │       Database
   │
   ├── mc01.vprofile.in
   │          │
   │          ▼
   │       Memcache
   │
   └── rmq01.vprofile.in
              │
              ▼
           RabbitMQ
```

This avoids making the application dependent on hardcoded backend IP addresses.

---

## 4. Application Artifact Deployment

The supplied application was built into a deployable WAR artifact and transferred through Amazon S3.

The deployment flow was:

```text
Application Source
       │
       ▼
     Maven
       │
       ▼
    WAR Artifact
       │
       ▼
      S3
       │
       ▼
Tomcat EC2 Instance
       │
       ▼
    Tomcat
       │
       ▼
Running Application
```

The artifact was built on the local machine and stored in S3 before being retrieved by the Tomcat instance.

This creates a separation between:

* build environment
* artifact storage
* runtime environment

---

## 5. IAM Access

Different authentication mechanisms were used for different parts of the artifact flow.

The local machine required AWS credentials to interact with S3.

The EC2 deployment environment used an IAM role to obtain AWS permissions without embedding long-lived AWS credentials into the instance.

The intended pattern was:

```text
Local Machine
     │
     │ AWS credentials
     ▼
    S3
     ▲
     │ IAM role
     │
Tomcat EC2
```

Secrets and access keys are not part of this repository.

---

## 6. Application Load Balancer

Configured an Application Load Balancer as the public entry point for the application.

The ALB receives client traffic and forwards it to the Tomcat application tier through a Target Group.

The traffic flow is:

```text
Client
  │
  │ HTTP / HTTPS
  ▼
 ALB
  │
  │ Target Group
  ▼
Tomcat :8080
```

The Target Group was configured for the port on which Tomcat serves the application.

---

## 7. Health Checks

Configured Target Group health checks for the Tomcat application instances.

The health-check flow provides an application-level signal to the load balancer:

```text
ALB
 │
 │ health check
 ▼
Tomcat :8080
 │
 ├── healthy → eligible for traffic
 │
 └── unhealthy → removed from traffic
```

This is more meaningful than simply checking whether the underlying EC2 instance is in a running state because the application service itself must respond correctly.

---

## 8. HTTPS

Configured HTTPS on the Application Load Balancer using an AWS Certificate Manager certificate.

The public traffic path becomes:

```text
Browser
   │
   │ HTTPS :443
   ▼
Application Load Balancer
   │
   │ HTTP :8080
   ▼
Tomcat
```

TLS terminates at the load balancer.

The backend application traffic from the ALB to Tomcat uses the configured HTTP target port.

---

## 9. DNS Mapping

The AWS load balancer provides an AWS-generated DNS endpoint.

A custom DNS name was mapped to the load balancer endpoint through the domain's DNS configuration.

The resulting flow is:

```text
User
 │
 │ https://<application-domain>
 ▼
DNS
 │
 ▼
ALB DNS endpoint
 │
 ▼
Application Load Balancer
```

This provides a stable, human-readable application endpoint instead of exposing the EC2 instance directly.

---

# Application-Tier Auto Scaling

After the fixed environment was validated, the Tomcat application tier was transitioned to Auto Scaling.

The scaling architecture is:

```text
                 AMI
                  │
                  ▼
          Launch Template
                  │
                  ▼
          Auto Scaling Group
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
       Tomcat  Tomcat  Tomcat
          │       │       │
          └───────┼───────┘
                  │
                  ▼
             Target Group
                  │
                  ▼
                 ALB
```

The AMI captures the configured application-instance state.

The Launch Template defines how new instances are created.

The Auto Scaling Group manages the desired application-tier capacity and connects the launched instances to the load balancer Target Group.

---

## Why Auto Scaling Was Applied to the Application Tier

The Auto Scaling Group was applied to the Tomcat application tier rather than the backend services.

The application tier is the component receiving user traffic through the load balancer and is therefore the natural tier for horizontal instance scaling in this architecture.

The database, cache, and message broker remained individually managed backend services.

This means the project demonstrates **application-tier elasticity**, not horizontal scaling of the entire platform.

---

## Auto Scaling Capacity

The project configuration used:

```text
Minimum capacity : 1
Desired capacity : 1
Maximum capacity : 4
```

This establishes the following relationship:

```text
1 ≤ desired capacity ≤ 4
```

The Auto Scaling Group can therefore maintain the application tier within the configured capacity boundaries.

---

## Target Tracking

The project uses target-tracking scaling based on application-instance resource utilization.

The demonstrated configuration uses CPU utilization as the scaling metric with a target around 50%.

Conceptually:

```text
CPU increases
     │
     ▼
Target exceeded
     │
     ▼
Scale out
     │
     ▼
Additional Tomcat instance
```

and:

```text
CPU decreases
     │
     ▼
Target below threshold
     │
     ▼
Scale in
```

The Auto Scaling Group remains bounded by its configured minimum and maximum capacity.

---

# Auto Scaling and Health

The Auto Scaling Group can use load-balancer health information in addition to EC2 instance health.

This distinction matters:

```text
EC2 health
    =
Is the instance running?

ELB health
    =
Is the application responding correctly?
```

By integrating the Target Group health state with the Auto Scaling Group, an unhealthy application instance can be replaced by the Auto Scaling mechanism.

The resulting lifecycle is:

```text
Instance launched
      │
      ▼
Registered with Target Group
      │
      ▼
Health check
      │
 ┌────┴────┐
 ▼         ▼
Healthy   Unhealthy
 │         │
 ▼         ▼
Traffic   Replacement
           instance
```

---

# End-to-End Validation

Validation was performed at multiple layers rather than relying only on the AWS console.

## Infrastructure Validation

Verified that the required AWS resources were available and correctly connected.

Examples included:

* EC2 instances
* Security Groups
* Route 53 records
* S3 artifact storage
* Application Load Balancer
* Target Group
* Auto Scaling Group

---

## DNS Validation

Verified that backend service names resolved correctly inside the VPC.

Examples:

```text
db01.vprofile.in
mc01.vprofile.in
rmq01.vprofile.in
```

The expected result was resolution to the corresponding backend private IP addresses.

---

## Target Group Validation

Verified that the application instance registered with the Target Group and reached a healthy state.

A healthy target establishes the following chain:

```text
Auto Scaling Group
        │
        ▼
Application Instance
        │
        ▼
Tomcat
        │
        ▼
Target Group Health Check
        │
        ▼
Healthy
        │
        ▼
ALB can route traffic
```

---

## Application Validation

Accessed the application through the load balancer rather than relying on direct application-instance access.

The final user-facing path was:

```text
Browser
   │
   ▼
DNS
   │
   ▼
ALB
   │
   ▼
Target Group
   │
   ▼
Tomcat
   │
   ▼
vProfile Application
```

---

## Backend Connectivity Validation

The application was used to validate its backend dependencies.

The validation covered:

### Database

Successful application authentication provided evidence that the application could communicate with the database.

### RabbitMQ

Application behavior was used to verify RabbitMQ connectivity.

### Memcache

Application behavior was used to verify caching.

The cache validation followed the pattern:

```text
First request
     │
     ▼
Database
     │
     ▼
Cache populated

Second request
     │
     ▼
Cache
```

This provided an application-level validation of the backend service integration rather than simply checking whether the backend EC2 instances were running.

---

# Deployment Flow

The overall deployment sequence was:

```text
1. AWS account
       │
       ▼
2. Key pair
       │
       ▼
3. Security Groups
       │
       ▼
4. EC2 instances
       │
       ▼
5. User-data initialization
       │
       ▼
6. Route 53 private DNS
       │
       ▼
7. Build application artifact
       │
       ▼
8. Upload artifact to S3
       │
       ▼
9. Download artifact to Tomcat
       │
       ▼
10. Configure ALB
       │
       ▼
11. Configure HTTPS / ACM
       │
       ▼
12. Configure public DNS
       │
       ▼
13. Validate application
       │
       ▼
14. Create AMI
       │
       ▼
15. Create Launch Template
       │
       ▼
16. Create Auto Scaling Group
       │
       ▼
17. Validate ASG-managed application instance
```

The Auto Scaling layer was deliberately added after the fixed deployment was working.

This follows the engineering progression:

> **Stabilize first, then automate.**

---

# Important Engineering Decisions

## 1. Lift and Shift Instead of Re-Architecture

The project intentionally preserved the existing application structure.

This reduced the scope of change and allowed the focus to remain on moving the workload to AWS infrastructure.

A future project can then address deeper cloud re-architecture.

---

## 2. Application Load Balancer Instead of Direct EC2 Access

Directly exposing the Tomcat instance would couple users to a specific instance.

The ALB provides a stable application entry point and allows the application tier to evolve behind it.

```text
Direct access:

User → EC2 :8080


ALB architecture:

User → ALB → Target Group → Tomcat
```

---

## 3. Security Group References Instead of Public Application Access

The application tier accepts application traffic from the load balancer security group rather than exposing Tomcat directly to the internet.

This establishes a clear trust relationship:

```text
Internet
   ↓
ALB SG
   ↓
App SG
   ↓
Backend SG
```

---

## 4. Route 53 Private DNS Instead of Backend IPs

The application configuration uses service names instead of directly embedding backend instance IP addresses.

This creates a level of indirection between:

```text
Application
     ↓
Service hostname
     ↓
Private DNS
     ↓
Backend IP
```

Replacing a backend instance therefore does not require changing the application's service hostname.

---

## 5. S3 as an Artifact Store

The application artifact is stored centrally in S3 before being deployed to the Tomcat environment.

This separates:

```text
Build
  ≠
Artifact storage
  ≠
Runtime
```

It also allows the same artifact to be retrieved by application instances created later.

---

## 6. IAM Role for EC2-to-AWS Access

The EC2 instance uses an IAM role for AWS service access rather than requiring long-lived access keys to be stored on the instance.

This follows the principle of using AWS-native identity mechanisms for AWS resources.

---

## 7. Auto Scaling Only After Validation

The application was first validated in a fixed configuration.

Only after the application, networking, DNS, load balancing, and backend connectivity were functioning was the application tier moved into Auto Scaling.

This reduced the number of moving parts during initial troubleshooting.

---

# Security Considerations

No credentials or secrets belong in this repository.

The following must never be committed:

* AWS access keys
* AWS secret access keys
* private SSH keys
* passwords
* authentication tokens
* certificates containing private material
* account-specific secrets
* environment-specific confidential information

Any screenshots published as evidence should be reviewed and sanitized before being committed.

Sensitive or environment-specific values should be replaced with placeholders where necessary.

---

# Evidence

Evidence is intentionally limited to high-signal artifacts.

The repository should contain only screenshots from my own completed environment that materially support the project claims.

Recommended evidence includes:

### Architecture

Shows the final AWS component relationships.

### Application Load Balancer / Target Group

Demonstrates:

* ALB configuration
* listener configuration
* target registration
* target health

### Route 53

Demonstrates private service-discovery records.

### Auto Scaling Group

Demonstrates:

* ASG configuration
* desired/min/max capacity
* application-tier management
* target-group integration

### Application Validation

Demonstrates that the deployed application was accessible through the intended load-balancing path and that the backend services were functioning.

Evidence should support the engineering story rather than document every console click.

---

# Project Limitations

This project intentionally has several boundaries.

## Existing Application

The vProfile application was supplied as the workload.

Application business logic and Java application development were not part of this project.

---

## No Infrastructure as Code Repository

The AWS infrastructure was provisioned through the practical execution workflow.

This repository therefore does not claim Terraform, CloudFormation, or another dedicated Infrastructure-as-Code implementation.

---

## No CI/CD Pipeline

The project demonstrates a manual build and artifact deployment flow:

```text
Build
  ↓
S3
  ↓
Tomcat
```

It does not demonstrate an automated Jenkins, GitHub Actions, GitLab CI/CD, or equivalent pipeline.

---

## No Production Observability Stack

Monitoring, centralized logging, alerting, and observability were not implemented as part of this project.

---

## Backend Services Are Not Auto Scaled

The Auto Scaling implementation applies to the application tier.

The database, cache, and message broker remain individually managed backend services.

---

## No Real Production Data Migration

The project demonstrates application and infrastructure migration rather than migration of a real production dataset.

No production database migration or synchronization strategy was implemented.

---

## No Cloud-Native Re-Architecture

The application architecture remains largely unchanged.

The project demonstrates:

> **Lift and Shift / IaaS**

rather than:

> **Cloud-native re-architecture**

---

## No Production-Scale Performance Validation

The project validates functional integration and AWS configuration.

It does not establish production-scale performance, capacity, resilience, or cost characteristics.

---

# What This Project Demonstrates

This project demonstrates practical experience with:

* AWS infrastructure provisioning
* EC2
* Security Groups
* private DNS
* Route 53
* S3
* IAM
* AWS CLI
* Tomcat deployment
* Maven-based artifact generation
* Application Load Balancing
* Target Groups
* HTTPS
* ACM
* AMIs
* Launch Templates
* Auto Scaling Groups
* application health checks
* backend service connectivity
* end-to-end validation
* AWS resource lifecycle management

The strongest engineering capability demonstrated is the ability to understand and operate the relationships between these components as one system rather than treating them as isolated AWS services.

---

# What This Project Does Not Demonstrate

This project should not be interpreted as demonstrating:

* development of the vProfile application
* Java application development
* application business-logic design
* Terraform
* CloudFormation
* CI/CD
* Jenkins
* GitHub Actions
* Kubernetes
* Docker
* production monitoring
* database high availability
* database replication
* zero-downtime deployment
* blue/green deployment
* canary deployment
* production data migration
* full cloud-native architecture

These capabilities may be addressed in later projects.

---

# Future Improvements

The natural next evolution of this project would be to move from manual IaaS deployment toward more automated and cloud-native engineering.

Potential future improvements include:

## 1. Infrastructure as Code

Rebuild the AWS environment using Terraform.

```text
Manual AWS Configuration
          ↓
       Terraform
          ↓
Reproducible Infrastructure
```

---

## 2. CI/CD

Replace the manual build-and-deploy flow with an automated pipeline.

```text
Git Push
   ↓
CI Build
   ↓
Test
   ↓
Artifact
   ↓
Artifact Repository
   ↓
Deployment
```

---

## 3. Automated Image Building

Replace manually created application images with a repeatable image-building process.

---

## 4. Observability

Introduce:

* metrics
* centralized logging
* dashboards
* alerting
* application health visibility

---

## 5. Improved Secrets Management

Move sensitive configuration away from static credentials and environment-specific files toward managed secret-storage mechanisms.

---

## 6. Backend High Availability

Reconsider the architecture of the database, cache, and message-broker tiers so that the entire system—not only the application tier—can tolerate individual component failures.

---

## 7. Externalized Application Sessions

The current application behavior can require session affinity when multiple application instances are used.

A future architecture could externalize session state so that application instances remain interchangeable without relying on load-balancer stickiness.

---

## 8. Cloud-Native Re-Architecture

A later iteration could replace portions of the EC2-based architecture with managed AWS services.

The progression would then become:

```text
Lift & Shift
     ↓
Automated Infrastructure
     ↓
CI/CD
     ↓
Managed Services
     ↓
Cloud-Native Architecture
```

These are **future improvements only** and are not implemented in this project.

---

# Project Summary

The project can be summarized as:

```text
Existing vProfile Application
            │
            │ Lift & Shift
            ▼
       AWS Infrastructure
            │
    ┌───────┼────────┐
    │       │        │
   EC2    Route 53   S3
    │       │        │
    │       │     Artifact
    │       │      Storage
    │       │
    ▼       ▼
 Tomcat  Backend Services
    │
    ▼
   ALB
    │
    ▼
 HTTPS / Custom DNS
    │
    ▼
   Users

          +
          
 AMI → Launch Template → Auto Scaling Group
```

The engineering focus was not the application itself.

It was the infrastructure and operational system required to take an existing multi-tier workload and run it on AWS with:

* separated infrastructure tiers
* controlled network access
* private service discovery
* artifact-based deployment
* secure public ingress
* application-tier elasticity
* health-aware traffic routing
* end-to-end validation

---

# Repository Structure

The public repository intentionally remains small:

```text
aws-vprofile-lift-and-shift/
│
├── README.md
├── architecture.png
│
└── evidence/
    └── screenshots/
```

### `README.md`

Primary project narrative, engineering memory, ownership boundary, implementation summary, validation, limitations, and navigation.

### `architecture.png`

Single high-level architecture diagram showing the relationships between the public entry point, application tier, backend services, DNS, and scaling components.

### `evidence/`

Contains only high-signal screenshots from the completed environment that support important project claims.

The supplied application source code and course-provided provisioning scripts are intentionally not reproduced in this repository.

---

# Final Takeaway

This project represents a practical **AWS lift-and-shift infrastructure deployment** around an existing multi-tier application.

The key engineering contribution was taking the workload from:

```text
Local VMs
    ↓
Manually hosted multi-tier application
```

to:

```text
AWS
 │
 ├── EC2-based service tiers
 ├── Layered Security Groups
 ├── Route 53 Private DNS
 ├── S3 artifact storage
 ├── IAM-based AWS access
 ├── Application Load Balancer
 ├── HTTPS / ACM
 ├── Target Group health checks
 └── Auto Scaling application tier
```

The project establishes practical AWS and DevOps infrastructure experience while maintaining a clear boundary between the **supplied application** and the **engineering work performed around that application**.

It is intentionally presented as a lift-and-shift infrastructure project rather than being represented as application development, Infrastructure as Code, CI/CD, or cloud-native re-architecture.
