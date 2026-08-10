# Architecture

[← Back to README](../README.md)

---

## 1. Architecture Overview

This project uses a **lift-and-shift architecture** to run an existing multi-tier application on AWS.

The application architecture is intentionally preserved rather than re-engineered.

The AWS environment separates the system into:

1. Public traffic entry
2. Application tier
3. Backend service tier

```text
                              Internet
                                 │
                                 │ HTTPS :443
                                 ▼
                    ┌─────────────────────────┐
                    │ Application Load        │
                    │ Balancer                │
                    │                         │
                    │ Public Entry Point      │
                    └────────────┬────────────┘
                                 │
                                 │ HTTP :8080
                                 ▼
                    ┌─────────────────────────┐
                    │ Application Tier        │
                    │                         │
                    │ Apache Tomcat           │
                    │ Auto Scaling Group       │
                    └────────────┬────────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  │              │              │
                  │              │              │
                  ▼              ▼              ▼
        ┌────────────────┐ ┌──────────────┐ ┌──────────────┐
        │ MySQL / MariaDB│ │   Memcache   │ │   RabbitMQ   │
        │                │ │              │ │              │
        │ Database Tier  │ │ Cache Tier   │ │ Broker Tier  │
        └────────────────┘ └──────────────┘ └──────────────┘
                  ▲              ▲              ▲
                  │              │              │
                  └──────────────┼──────────────┘
                                 │
                         Private DNS
                         Route 53 Hosted Zone
```

---

## 2. Architecture Layers

The environment can be understood as three major infrastructure layers.

### Public Entry Layer

The public entry layer contains the Application Load Balancer.

Its responsibilities are:

* receive application traffic
* provide the public application endpoint
* terminate HTTPS
* forward requests to healthy application targets

The application itself is not exposed directly to the public internet.

---

### Application Layer

The application layer contains the Tomcat application instances.

Its responsibilities are:

* run the supplied vProfile application
* receive traffic from the Application Load Balancer
* communicate with the backend services
* participate in the Auto Scaling Group

The application tier is the only tier in this project that is placed behind an Auto Scaling Group.

---

### Backend Service Layer

The backend layer contains:

* MySQL / MariaDB
* Memcache
* RabbitMQ

These services support the application but remain individually managed EC2-based services.

They are not exposed directly to the internet.

---

# 3. Application Ownership Boundary

The application running in this architecture is the **vProfile application supplied by the project/course material**.

The application source and business logic were not developed as part of this project.

The architectural contribution represented here is therefore:

```text
Supplied Application
        │
        ▼
AWS Infrastructure Engineering
        │
        ├── Infrastructure
        ├── Networking
        ├── Security
        ├── Service Discovery
        ├── Deployment
        ├── Load Balancing
        ├── HTTPS
        ├── Auto Scaling
        └── Validation
```

The architecture should therefore be understood as:

> **AWS infrastructure surrounding an existing application workload.**

---

# 4. AWS Infrastructure

The infrastructure uses Amazon EC2 instances for the application and backend service tiers.

The major EC2 roles are:

```text
Application
    │
    └── Tomcat

Database
    │
    └── MySQL / MariaDB

Cache
    │
    └── Memcache

Message Broker
    │
    └── RabbitMQ
```

Each service is separated logically through its own infrastructure and security boundaries.

The project therefore preserves the multi-tier structure of the original application while moving the infrastructure to AWS.

---

# 5. Security Boundaries

Security Groups are used to establish traffic boundaries between the tiers.

The intended trust relationship is:

```text
                         Internet
                            │
                            ▼
                    ┌──────────────┐
                    │     ALB SG   │
                    └──────┬───────┘
                           │
                         8080
                           │
                           ▼
                    ┌──────────────┐
                    │     App SG   │
                    └──────┬───────┘
                           │
                  Backend Service Ports
                           │
                           ▼
                    ┌──────────────┐
                    │  Backend SG  │
                    └──────────────┘
```

This creates three important boundaries:

### Internet → Load Balancer

Public application traffic reaches the Application Load Balancer.

### Load Balancer → Application

The application tier accepts application traffic from the load balancer rather than directly from arbitrary internet clients.

### Application → Backend

Backend services accept traffic from the application tier as required by the application.

This keeps the backend services outside the public application path.

---

# 6. Application Traffic Flow

The user-facing traffic path is:

```text
Client
  │
  │ HTTPS :443
  ▼
Application Load Balancer
  │
  │ Target Group
  │ HTTP :8080
  ▼
Tomcat Application Instance
  │
  ├──────────► MySQL / MariaDB
  │
  ├──────────► Memcache
  │
  └──────────► RabbitMQ
```

The Application Load Balancer therefore acts as the stable public entry point while the application instances remain behind it.

This separation is important once the application tier becomes dynamically managed by the Auto Scaling Group.

---

# 7. Application Load Balancer

The Application Load Balancer provides the public-facing application endpoint.

Its primary responsibilities are:

* accept client connections
* provide HTTP/HTTPS listeners
* terminate HTTPS
* forward requests to the Target Group
* perform target health checks
* route traffic only to healthy targets

The resulting relationship is:

```text
                    Application Load Balancer
                              │
                              ▼
                         Target Group
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              Tomcat Instance     Tomcat Instance
```

The second Tomcat instance becomes relevant when the Auto Scaling Group scales the application tier beyond one instance.

---

# 8. Target Group

The Target Group represents the application instances that can receive traffic from the load balancer.

The application is served by Tomcat on port `8080`.

The relationship is:

```text
ALB
 │
 │ HTTP :8080
 ▼
Target Group
 │
 ├── Tomcat instance
 ├── Tomcat instance
 └── ...
```

The Target Group also provides the health-check mechanism used to determine whether an application instance is healthy enough to receive traffic.

---

# 9. Health Model

There are two different concepts of health involved in the final architecture.

### EC2 Instance Health

This indicates whether the EC2 instance itself is functioning from the infrastructure perspective.

### Application / Target Health

The Target Group health check verifies whether the application service responds correctly through the configured application port.

Conceptually:

```text
EC2 running
     │
     ▼
Tomcat running
     │
     ▼
Target Group health check
     │
     ├── Healthy
     │      │
     │      ▼
     │   Eligible for traffic
     │
     └── Unhealthy
            │
            ▼
       Not eligible for traffic
```

The distinction becomes particularly important when Auto Scaling is enabled.

---

# 10. Private Service Discovery

The backend services are identified through Route 53 Private DNS names.

The application uses service names rather than depending directly on backend private IP addresses.

The conceptual mapping is:

```text
db01.vprofile.in
       │
       ▼
MySQL / MariaDB private IP


mc01.vprofile.in
       │
       ▼
Memcache private IP


rmq01.vprofile.in
       │
       ▼
RabbitMQ private IP
```

These records exist inside the private DNS namespace associated with the VPC.

---

# 11. Why Private DNS Is Used

The application needs stable names for its backend dependencies.

Using private DNS creates the following abstraction:

```text
Application
     │
     ▼
Service Name
     │
     ▼
Route 53 Private DNS
     │
     ▼
Current Backend IP
```

The application therefore does not need to directly depend on a specific backend instance IP in the architecture.

This is especially useful when infrastructure changes require backend instances to be replaced.

---

# 12. Artifact and Application Deployment Architecture

The application deployment uses an artifact-based flow.

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
   Tomcat EC2
        │
        ▼
     Tomcat
        │
        ▼
Running Application
```

This creates a separation between:

* application build
* artifact storage
* application runtime

The AWS environment therefore does not require the build process to occur directly on the production-style application instance.

---

# 13. IAM Architecture

The deployment uses different AWS access patterns for the local environment and EC2.

Conceptually:

```text
Local Machine
      │
      │ AWS credentials
      ▼
     S3
      ▲
      │ IAM Role
      │
Tomcat EC2
```

The important principle is that the EC2 instance can access AWS services using an attached IAM role rather than requiring long-lived AWS credentials to be stored on the instance.

Credentials and secrets are intentionally excluded from this repository.

---

# 14. HTTPS Architecture

HTTPS is terminated at the Application Load Balancer.

The public traffic flow is:

```text
Client
  │
  │ HTTPS :443
  ▼
ALB
  │
  │ HTTP :8080
  ▼
Tomcat
```

AWS Certificate Manager provides the certificate used by the HTTPS listener.

This allows the public application endpoint to use HTTPS while the ALB handles the TLS termination.

---

# 15. Public DNS Flow

The public application endpoint is represented by a custom DNS name that maps to the load balancer endpoint.

The flow is:

```text
User
 │
 │ Application Domain
 ▼
Public DNS
 │
 ▼
ALB DNS Name
 │
 ▼
Application Load Balancer
 │
 ▼
Target Group
 │
 ▼
Tomcat
```

The user therefore interacts with a stable application hostname rather than directly accessing an EC2 instance.

---

# 16. Auto Scaling Architecture

The application tier is transitioned from a manually managed EC2 instance into an Auto Scaling Group.

The architecture is:

```text
                     AMI
                      │
                      ▼
              Launch Template
                      │
                      ▼
              Auto Scaling Group
                      │
             ┌────────┼────────┐
             │        │        │
             ▼        ▼        ▼
          Tomcat   Tomcat   Tomcat
             │        │        │
             └────────┼────────┘
                      │
                      ▼
                 Target Group
                      │
                      ▼
                     ALB
```

---

# 17. AMI

An Amazon Machine Image is created from the configured application instance.

The AMI captures the instance state needed to reproduce the application server.

The conceptual flow is:

```text
Configured Tomcat Instance
          │
          ▼
         AMI
          │
          ▼
Reusable Application Instance Image
```

The AMI therefore becomes the basis for launching additional application instances.

---

# 18. Launch Template

The Launch Template defines how application instances are launched.

The relationship is:

```text
AMI
 │
 ▼
Launch Template
 │
 ├── Instance configuration
 ├── Security Group
 ├── IAM configuration
 └── Other launch settings
```

The Auto Scaling Group uses the Launch Template as the definition for new application instances.

---

# 19. Auto Scaling Group

The Auto Scaling Group manages the application-tier capacity.

The project configuration uses:

```text
Minimum capacity : 1
Desired capacity : 1
Maximum capacity : 4
```

Conceptually:

```text
Minimum
   │
   ▼
  [1] ← Desired
   │
   │ scale out
   ▼
 [2] → [3] → [4]
                 ▲
                 │
               Maximum
```

The Auto Scaling Group is therefore responsible for maintaining application-tier capacity within the configured limits.

---

# 20. Scaling Policy

The application tier uses target tracking based on CPU utilization.

The conceptual behavior is:

```text
CPU utilization increases
          │
          ▼
Target utilization exceeded
          │
          ▼
Application tier scales out
```

and:

```text
CPU utilization decreases
          │
          ▼
Target utilization falls
          │
          ▼
Application tier can scale in
```

The Auto Scaling Group remains constrained by the configured minimum and maximum capacity.

---

# 21. Self-Healing Relationship

The final architecture connects the Auto Scaling Group with the load balancer's health information.

The lifecycle can be represented as:

```text
ASG launches instance
        │
        ▼
Instance registered with Target Group
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
allowed   instance
```

This provides a degree of self-healing for the application tier.

The scope of this behavior should not be interpreted as self-healing for the entire application stack because the backend services remain individually managed.

---

# 22. Why Only the Application Tier Scales

The project applies Auto Scaling to the Tomcat application tier.

The reason is architectural:

```text
                    User Traffic
                         │
                         ▼
                        ALB
                         │
                         ▼
                 Application Tier
                         │
                ┌────────┼────────┐
                ▼        ▼        ▼
              DB       Cache     Broker
```

The application tier directly receives user traffic and is therefore the natural tier for horizontal scaling in this implementation.

The backend services have different state and lifecycle characteristics and remain individually managed in this project.

---

# 23. Final Traffic and Dependency Model

The complete system can be represented as:

```text
                              Internet
                                 │
                                 │ HTTPS
                                 ▼
                       ┌───────────────────┐
                       │       ALB         │
                       └─────────┬─────────┘
                                 │
                              :8080
                                 │
                                 ▼
                       ┌───────────────────┐
                       │  Auto Scaling     │
                       │  Application Tier │
                       │      Tomcat       │
                       └─────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        MySQL / MariaDB       Memcache          RabbitMQ
              ▲                  ▲                  ▲
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                         Route 53 Private DNS
```

The major relationships are therefore:

```text
Public DNS
    ↓
Application Load Balancer
    ↓
Target Group
    ↓
Auto Scaling Group
    ↓
Tomcat
    ↓
Private DNS
    ↓
Backend Services
```

---

# 24. Important Architecture Decisions

## Lift & Shift

The application architecture was preserved instead of being re-engineered.

**Reason:** keep the migration focused on moving the existing workload to AWS.

---

## Application Load Balancer

The ALB provides a stable public entry point.

**Reason:** avoid coupling users directly to an individual application instance and allow the application tier to scale behind the load balancer.

---

## Layered Security Groups

Traffic is controlled between the major infrastructure layers.

**Reason:** prevent unnecessary direct access between unrelated tiers.

---

## Private Route 53

Backend services are addressed through internal DNS names.

**Reason:** provide service-name-based discovery instead of coupling the application directly to backend IP addresses.

---

## S3 Artifact Storage

The WAR artifact passes through S3 before deployment.

**Reason:** separate build output from the application runtime environment.

---

## IAM Role

EC2 accesses AWS resources through an IAM role.

**Reason:** avoid storing long-lived AWS credentials on the application instance.

---

## AMI + Launch Template + ASG

The application instance is converted into a reusable launch definition and managed by an Auto Scaling Group.

**Reason:** make the application tier reproducible and capable of automatic scaling and instance replacement.

---

# 25. Architecture Boundary

This architecture demonstrates:

* AWS IaaS deployment
* multi-tier infrastructure
* layered network security
* private service discovery
* artifact-based application deployment
* load balancing
* HTTPS
* application-tier Auto Scaling
* health-aware traffic routing
* application/backend integration

It does **not** establish:

* cloud-native architecture
* Infrastructure as Code
* CI/CD
* Kubernetes
* Docker
* production monitoring
* backend high availability
* production data migration
* production-scale performance characteristics

The architecture should therefore be understood as a **lift-and-shift AWS deployment**, not as a final production cloud architecture.

---

[← Back to README](../README.md)
[Implementation →](implementation.md)
[Validation →](validation.md)
