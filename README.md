# AWS Lift & Shift — vProfile Multi-Tier Application

AWS lift-and-shift deployment of an existing multi-tier Java application using **Amazon EC2, Route 53, S3, IAM, Application Load Balancer, ACM, and Auto Scaling**.

> **Focus:** AWS infrastructure, application deployment, service connectivity, secure ingress, application-tier scaling, and end-to-end validation.

---

## Overview

This project demonstrates the migration of an existing multi-tier application from a local virtual-machine environment to AWS using a **lift-and-shift** approach.

The application architecture was preserved rather than re-engineered. The engineering work focused on the AWS infrastructure and operational configuration required to deploy, expose, scale, and validate the existing workload.

### Final deployment

```text
                         Internet
                            │
                         HTTPS
                            │
                            ▼
                 ┌────────────────────┐
                 │ Application Load   │
                 │ Balancer           │
                 └─────────┬──────────┘
                           │
                        HTTP :8080
                           │
                           ▼
                 ┌────────────────────┐
                 │ Application Tier   │
                 │ Tomcat + ASG       │
                 └─────────┬──────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
          MySQL /      Memcache      RabbitMQ
           MariaDB
              ▲            ▲            ▲
              └────────────┼────────────┘
                           │
                    Route 53 Private DNS
```

For the detailed architecture and design rationale, see **[Architecture](docs/architecture.md)**.

---

## Application Ownership

The **vProfile application was used as the workload for this project and was not developed as part of this work**.

The application source, business logic, and course-provided provisioning artifacts originated from the supplied project material.

My engineering work focused on the AWS environment required to run the existing workload, including:

* AWS infrastructure provisioning
* network and security configuration
* private service discovery
* application artifact deployment
* load balancing
* HTTPS
* application-tier Auto Scaling
* health validation
* end-to-end validation
* AWS resource cleanup

This repository therefore represents the **AWS engineering performed around the application**, rather than development of the application itself.

---

## Engineering Work

| Area           | Work                                                   |
| -------------- | ------------------------------------------------------ |
| Infrastructure | EC2-based multi-tier deployment                        |
| Networking     | Security Groups and Route 53 Private DNS               |
| Deployment     | Maven → WAR → S3 → Tomcat                              |
| Traffic        | Application Load Balancer + Target Group               |
| Security       | IAM + Security Groups + HTTPS                          |
| Scaling        | AMI → Launch Template → Auto Scaling Group             |
| Validation     | DNS, health checks, application and backend validation |

See **[Implementation](docs/implementation.md)** for the detailed engineering flow.

---

## Key Engineering Decisions

### Lift & Shift

The existing application architecture was preserved instead of being re-engineered. This kept the project focused on migrating the workload to AWS infrastructure.

### Layered Security Groups

Traffic was separated between the public load-balancing layer, application tier, and backend services.

### Private DNS

Route 53 Private DNS was used for backend service discovery rather than making the application depend directly on backend instance IP addresses.

### S3 Artifact Storage

The application WAR artifact was transferred through S3 before being deployed to the Tomcat application tier.

### Application Load Balancer

The ALB provides the public application entry point and routes traffic to the Tomcat Target Group.

### Application-Tier Auto Scaling

The Tomcat application tier was placed behind an Auto Scaling Group using an AMI and Launch Template.

The backend services remained individually managed.

---

## Validation

The final environment was validated through:

* EC2 and infrastructure checks
* private DNS resolution
* Target Group health checks
* application accessibility through the ALB
* database connectivity
* RabbitMQ connectivity
* Memcache behavior
* ASG-managed application instance validation

See **[Validation](docs/validation.md)** for the detailed validation approach.

High-signal execution evidence is maintained under [`evidence/screenshots/`](evidence/screenshots/).

---

## Project Boundaries

This project demonstrates **AWS lift-and-shift infrastructure engineering**.

It does **not** demonstrate:

* development of the vProfile application
* Java application development
* Terraform / Infrastructure as Code
* CI/CD
* Jenkins
* GitHub Actions
* Kubernetes
* Docker
* production monitoring
* database high availability
* production data migration
* cloud-native re-architecture

These boundaries are intentional and prevent the project from overstating its scope.

See **[Limitations & Future Work](docs/limitations-and-future-work.md)** for details and potential next steps.

---

## Technologies

**AWS:** EC2 · Route 53 · S3 · IAM · Application Load Balancer · Target Groups · ACM · Auto Scaling · AMI · Launch Templates · AWS CLI

**Application:** Maven · Tomcat · MySQL/MariaDB · RabbitMQ · Memcache

---

## Repository Structure

```text
aws-vprofile-lift-and-shift/
│
├── README.md
├── architecture.png
│
├── docs/
│   ├── architecture.md
│   ├── implementation.md
│   ├── validation.md
│   └── limitations-and-future-work.md
│
└── evidence/
    └── screenshots/
```

### Documentation

* **[Architecture](docs/architecture.md)** — system architecture, traffic flows, security boundaries, service discovery, scaling design, and engineering rationale.
* **[Implementation](docs/implementation.md)** — infrastructure provisioning, deployment flow, ALB, HTTPS, DNS, AMI, Launch Template, and Auto Scaling implementation.
* **[Validation](docs/validation.md)** — infrastructure, DNS, Target Group, application, backend, and Auto Scaling validation.
* **[Limitations & Future Work](docs/limitations-and-future-work.md)** — current project boundaries and potential future evolution.

### Evidence

The `evidence/screenshots/` directory contains only high-signal evidence from the completed environment that supports important project claims.

---

## Project Summary

The project moves an existing multi-tier application from a local VM-based environment to AWS:

```text
Existing Application
        │
        │ Lift & Shift
        ▼
AWS Infrastructure
        │
        ├── EC2
        ├── Security Groups
        ├── Route 53 Private DNS
        ├── S3
        ├── IAM
        ├── Application Load Balancer
        ├── ACM / HTTPS
        └── Auto Scaling
```

The strongest engineering capability demonstrated is the ability to configure and operate these AWS components as an integrated system:

**Infrastructure → Networking → Security → Deployment → Load Balancing → HTTPS → Scaling → Validation**

The project intentionally stops at the demonstrated lift-and-shift scope rather than claiming Infrastructure as Code, CI/CD, production monitoring, or cloud-native re-architecture.
