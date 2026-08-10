# Implementation

[← Back to README](../README.md) · [← Architecture](architecture.md)

---

## 1. Implementation Approach

The project was implemented as a staged **lift-and-shift deployment**.

The existing multi-tier application was moved from a local VM-based environment to AWS while keeping the application's fundamental service structure intact.

The implementation deliberately followed this sequence:

```text
AWS Account
    ↓
Key Pair
    ↓
Security Groups
    ↓
EC2 + User Data
    ↓
Private DNS
    ↓
Build Application
    ↓
Upload Artifact to S3
    ↓
Deploy Artifact to Tomcat
    ↓
Application Load Balancer
    ↓
HTTPS / ACM
    ↓
Public DNS
    ↓
End-to-End Validation
    ↓
Auto Scaling Group
```

The Auto Scaling Group was added **after the fixed deployment had been verified**.

This follows the project's operational principle:

> **Stabilize first, then automate.**

---

# 2. Infrastructure Components

The AWS implementation uses EC2 instances for the application and backend services.

The logical service layout is:

```text
Application Tier
    └── Tomcat

Backend Services
    ├── MySQL / MariaDB
    ├── Memcache
    └── RabbitMQ
```

The Application Load Balancer provides the public entry point.

Route 53 provides private service discovery.

S3 provides the intermediary artifact storage layer.

Auto Scaling manages the Tomcat application tier.

---

# 3. Step 1 — AWS Access

The AWS Management Console was used as the starting point for provisioning the environment.

The AWS account provides the environment in which the required infrastructure resources are created and managed.

---

# 4. Step 2 — EC2 Key Pair

An EC2 key pair was created for SSH access to the instances.

The private key is used to authenticate when connecting to the EC2 instances for tasks such as:

* configuration
* troubleshooting
* artifact deployment
* validation

The private key is environment-specific and must not be committed to Git.

The source material also emphasizes that the private key is available only at key-pair creation time and therefore needs to be stored securely.

---

# 5. Step 3 — Security Groups

Three security groups were created to represent the major infrastructure layers:

```text
SG1 — Load Balancer
SG2 — Tomcat Application Tier
SG3 — Backend Services
```

The intended traffic chain is:

```text
Internet
   │
   │ HTTPS :443
   ▼
SG1 — Load Balancer
   │
   │ HTTP :8080
   ▼
SG2 — Tomcat
   │
   │ Backend service ports
   ▼
SG3 — Backend Services
```

---

## 5.1 Load Balancer Security Group

The load balancer security group permits public HTTPS traffic.

Conceptually:

```text
Inbound:
443 ← Internet
```

The public-facing security boundary therefore exists at the ALB rather than at the Tomcat instances.

---

## 5.2 Tomcat Security Group

The Tomcat security group permits application traffic on port `8080` only from the load balancer security group.

Conceptually:

```text
Inbound:
8080 ← Load Balancer Security Group
```

The source material specifically uses a security-group reference rather than an unrestricted IP range for this relationship.

---

## 5.3 Backend Security Group

The backend security group permits the required service ports only from the Tomcat security group.

Conceptually:

```text
Inbound:
Database ports  ← Tomcat SG
Cache ports     ← Tomcat SG
Broker ports    ← Tomcat SG
```

This creates the three-layer security chain:

```text
Internet
   ↓
Load Balancer SG
   ↓
Tomcat SG
   ↓
Backend SG
```

The model follows a defense-in-depth approach in which each layer is reachable only from the layer directly above it.

---

# 6. Step 4 — EC2 Instances

The required EC2 instances were launched using the appropriate operating system images, instance configuration, key pair, and security group.

The major roles were:

```text
app01
  └── Tomcat

db01
  └── MySQL / MariaDB

mc01
  └── Memcache

rmq01
  └── RabbitMQ
```

The instances were configured using user-data scripts where applicable.

---

# 7. User Data and Instance Initialization

User Data was used to automate the initial software configuration performed when the EC2 instances were launched.

The source material describes separate provisioning scripts for the service roles.

The important architectural purpose of User Data in this project is:

```text
EC2 Launch
    ↓
User Data
    ↓
Initial software configuration
    ↓
Service available
```

This allows a newly launched instance to be initialized without repeating the entire installation process manually.

---

## 7.1 Tomcat Initialization

The Tomcat instance uses the Ubuntu-based Tomcat provisioning approach.

The source material describes the Tomcat user-data script as intentionally simple, with the package manager handling:

* Tomcat installation
* service creation
* service registration

Application artifact deployment was deliberately handled later through the S3-based deployment flow rather than being embedded into the initial Tomcat provisioning.

---

## 7.2 Backend Initialization

The backend instances were initialized according to their service roles.

The RabbitMQ provisioning process, for example, installs its required dependencies, repository configuration, Erlang, RabbitMQ, and service configuration before enabling the service.

The specific course-provided provisioning scripts are **not reproduced in this repository** because they are not original portfolio artifacts.

---

# 8. Instance Validation

Before continuing with the application deployment, the EC2 instances and their services need to be operational.

The validation target is:

```text
db01 → MySQL / MariaDB running
mc01 → Memcache running
rmq01 → RabbitMQ running
app01 → Tomcat running
```

The practical also establishes a useful operational principle for the provisioning stage:

> If a service instance is incorrectly initialized, recreate the instance using the correct AMI, security group, and User Data rather than accumulating manual configuration changes.

This keeps the infrastructure closer to the intended reproducible initialization model.

---

# 9. Step 5 — Route 53 Private DNS

Once the backend instances existed, their private IP addresses were available.

A Route 53 Private Hosted Zone was associated with the VPC.

DNS records were then created for the backend services.

The conceptual mapping is:

```text
db01.vprofile.in
        ↓
Database private IP


mc01.vprofile.in
        ↓
Memcache private IP


rmq01.vprofile.in
        ↓
RabbitMQ private IP
```

The application therefore uses DNS names rather than hardcoded backend IP addresses.

---

## 9.1 DNS Validation

DNS resolution was validated from within the VPC.

For example:

```bash
nslookup db01.vprofile.in
```

The expected result is resolution to the private IP address of the corresponding backend instance.

The same approach can be used for the cache and message-broker records.

The Private Hosted Zone must be associated with the correct VPC; otherwise the internal DNS names will not resolve from the application instances.

---

# 10. Application Configuration

The application configuration uses the private DNS names created through Route 53.

The relevant dependency pattern is:

```text
application.properties
        │
        ├── db01.vprofile.in
        ├── mc01.vprofile.in
        └── rmq01.vprofile.in
                │
                ▼
        Route 53 Private DNS
                │
                ▼
        Backend private IPs
```

The source material specifically notes that incorrect hostnames can allow the application to build successfully while still causing runtime failure.

Therefore the application configuration needs to be verified before building the deployable artifact.

---

# 11. Step 6 — Build the Application

The application source was built on the local machine rather than directly on the Tomcat server.

The build flow is:

```text
Application Source
       ↓
     Maven
       ↓
   WAR Artifact
```

The source material specifies the local build toolchain as:

* Java 17
* Maven 3.9.9
* AWS CLI

The resulting WAR file is the deployable application artifact.

---

# 12. Build / Runtime Separation

The implementation deliberately separates the build environment from the runtime environment.

```text
Build Environment
      │
      │ WAR
      ▼
Artifact Storage
      │
      │
      ▼
Runtime Environment
```

The local machine performs the build.

The Tomcat EC2 instance runs the application.

This prevents the runtime environment from becoming the application build environment.

---

# 13. Step 7 — Upload Artifact to S3

The compiled WAR artifact is uploaded from the local machine to an S3 bucket.

The deployment flow is:

```text
Local Machine
      │
      │ upload
      ▼
S3 Bucket
```

S3 acts as the intermediary artifact store.

This creates a clean separation between the machine that builds the application and the machine that runs it.

---

# 14. Artifact Pipeline

The complete artifact flow is:

```text
             BUILD
               │
               ▼
       ┌────────────────┐
       │ Local Machine  │
       │ Maven + JDK    │
       └───────┬────────┘
               │
               │ WAR
               ▼
             STORE
               │
               ▼
       ┌────────────────┐
       │   S3 Bucket    │
       └───────┬────────┘
               │
               │ download
               ▼
            DEPLOY
               │
               ▼
       ┌────────────────┐
       │ Tomcat EC2     │
       │                │
       │ /webapps/      │
       └────────────────┘
```

This is the project's **Build → Store → Deploy** pattern.

The same architectural pattern can later be automated through CI/CD systems, but CI/CD itself was not implemented in this project.

---

# 15. IAM for Artifact Access

The artifact pipeline uses two AWS access patterns.

### Local machine → S3

The local machine uses AWS credentials configured for AWS CLI access.

### EC2 → S3

The EC2 instance uses an IAM role for access to S3.

The conceptual model is:

```text
Local Machine
     │
     │ credentials
     ▼
    S3
     ▲
     │ IAM Role
     │
Tomcat EC2
```

This avoids storing long-lived AWS access keys on the application server.

---

# 16. Credential Security

AWS credentials are treated as sensitive information.

The following must not be committed:

```text
AWS access keys
AWS secret keys
SSH private keys
Passwords
Tokens
Private certificates
```

The source material explicitly warns that exposed AWS keys can result in unauthorized resource usage and unexpected charges.

Any local credential files should therefore remain outside the repository.

---

# 17. Step 8 — Deploy Artifact to Tomcat

The WAR artifact is downloaded from S3 to the Tomcat EC2 instance.

The Tomcat deployment flow is:

```text
S3
 │
 │ AWS CLI download
 ▼
Tomcat EC2
 │
 ▼
/var/lib/tomcat10/webapps/
 │
 ▼
ROOT.war
 │
 ▼
Tomcat
 │
 ▼
Application
```

The WAR is renamed to `ROOT.war` so that Tomcat serves the application at the root path `/`.

The Tomcat deployment directory is:

```text
/var/lib/tomcat10/webapps/
```

Tomcat detects `ROOT.war` and extracts it into the corresponding `ROOT/` application directory.

---

# 18. Tomcat Deployment Validation

After deploying the artifact, the Tomcat service must be verified.

The important checks are:

```bash
systemctl status tomcat10
```

and confirmation that the application has been extracted into the Tomcat `webapps` directory.

The source material also notes that Tomcat may need a short amount of time after startup to extract the WAR.

---

# 19. Step 9 — Application Load Balancer

Once the application is working on the Tomcat instance, an Application Load Balancer is introduced.

The traffic path becomes:

```text
Client
  │
  ▼
Application Load Balancer
  │
  ▼
Target Group
  │
  ▼
Tomcat :8080
```

The ALB replaces the direct application-instance access pattern with a managed public entry point.

---

# 20. Target Group

The Tomcat application instance is registered with an ALB Target Group.

The Target Group defines:

* application target
* target port
* health-check behavior

The application runs on Tomcat port `8080`.

Therefore:

```text
ALB
 │
 │ Target Group
 │
 └──► Tomcat :8080
```

The Target Group is also used later by the Auto Scaling Group.

---

# 21. Target Health

The ALB health check validates the application target.

The basic relationship is:

```text
Target Group
      │
      ▼
Health Check
      │
      ▼
Tomcat
```

A healthy target is eligible to receive traffic.

An unhealthy target is removed from normal load-balancing traffic.

This health information becomes important later when the Target Group is connected to the Auto Scaling Group.

---

# 22. Step 10 — HTTPS with ACM

HTTPS was configured on the Application Load Balancer.

AWS Certificate Manager provides the certificate.

The public traffic flow becomes:

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

The ALB therefore acts as the TLS termination point.

The certificate is attached to the ALB's HTTPS listener.

---

# 23. Public DNS

After the ALB was configured, its AWS-generated DNS name became available.

A public DNS CNAME record was configured to map the desired application hostname to the ALB endpoint.

The flow becomes:

```text
Application Domain
        │
        ▼
Public DNS
        │
        ▼
ALB DNS Name
        │
        ▼
Application Load Balancer
```

The source material describes using GoDaddy DNS management for this CNAME mapping.

---

# 24. Step 11 — End-to-End Validation

Before introducing Auto Scaling, the complete static deployment was validated.

The validation chain is:

```text
User
 │
 ▼
Public DNS
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
 ├──► MySQL / MariaDB
 │
 ├──► Memcache
 │
 └──► RabbitMQ
```

The validation confirms that the individual components are not only running but can work together as one application system.

The source material explicitly describes this as the stage where the user URL, ALB, Tomcat, Route 53, and backend services are validated together.

---

# 25. Application-Level Backend Validation

The application itself provides an important validation mechanism.

### Database

Successful application authentication demonstrates that the application can communicate with the database.

### RabbitMQ

Application behavior is used to validate RabbitMQ connectivity.

### Memcache

Application behavior can demonstrate cache usage.

This is stronger than checking only:

```text
EC2 = running
```

because the application must actually communicate with the backend dependencies.

---

# 26. Stabilize Before Automating

The Auto Scaling Group is deliberately not created immediately.

The implementation sequence is:

```text
Build static deployment
        ↓
Validate infrastructure
        ↓
Validate DNS
        ↓
Validate ALB
        ↓
Validate application
        ↓
Validate backend connectivity
        ↓
Only then enable Auto Scaling
```

This prevents Auto Scaling from multiplying an incorrectly configured application instance.

The source material explicitly describes this as:

> **Stabilize first, then automate.**

---

# 27. Step 12 — Create AMI

Once the application instance is fully configured and validated, an AMI is created from the configured application instance.

The AMI provides the reusable base for future application instances.

```text
Configured app01
       │
       ▼
      AMI
       │
       ▼
Reusable Tomcat Instance
```

The purpose is to capture the known-good application-instance state before moving to Auto Scaling.

---

# 28. Create Launch Template

A Launch Template is created using the application AMI.

The Launch Template defines how future application instances should be created.

The configuration includes the required instance characteristics such as:

* AMI
* instance type
* security group
* key pair
* instance configuration
* user-data configuration where required

The relationship is:

```text
AMI
 │
 ▼
Launch Template
 │
 ▼
Future Application Instances
```

---

# 29. Create Auto Scaling Group

The Auto Scaling Group is created using the Launch Template.

The source material specifies the following configuration:

```text
Desired capacity : 1
Minimum capacity : 1
Maximum capacity : 4
```

The ASG is also associated with the existing Application Load Balancer Target Group.

The resulting architecture is:

```text
Launch Template
       │
       ▼
Auto Scaling Group
       │
       ├── Tomcat instance
       ├── Tomcat instance
       ├── Tomcat instance
       └── Tomcat instance
              │
              ▼
         Target Group
              │
              ▼
             ALB
```

Only the number of running application instances changes.

The application service itself remains the same.

---

# 30. Availability Zones

The Auto Scaling Group is configured across the available Availability Zones selected for the VPC.

This allows instances launched by the group to be distributed across zones according to the configured ASG placement behavior.

The source material specifically selects all available Availability Zones during the ASG setup.

---

# 31. Load Balancer Integration

The Auto Scaling Group is attached to the existing Target Group.

This creates an automatic registration relationship:

```text
ASG launches instance
        │
        ▼
Instance automatically registered
        │
        ▼
Target Group
        │
        ▼
ALB
```

New application instances therefore become part of the traffic path without requiring manual Target Group registration.

---

# 32. ELB Health Checks

Elastic Load Balancing health checks are enabled for the Auto Scaling Group.

This is important because an EC2 instance being in a `running` state does not necessarily mean the Tomcat application is healthy.

The resulting model is:

```text
EC2 health
    +
ELB / Target Group health
    ↓
ASG lifecycle decision
```

If the application target becomes unhealthy, the ASG can terminate and replace the affected instance.

The source material explicitly highlights this distinction and the role of ELB health checks in application-instance replacement.

---

# 33. Scaling Policy

The Auto Scaling Group uses a **Target Tracking Scaling Policy**.

The configured metric is:

```text
CPU Utilization
```

The target value is:

```text
50%
```

The intended behavior is:

```text
Average CPU > target
        ↓
Scale out


Average CPU < target
        ↓
Scale in
```

The ASG remains constrained by:

```text
Minimum = 1
Desired = 1
Maximum = 4
```

---

# 34. Replacing the Original Application Instance

After the Auto Scaling Group launches its own application instance from the AMI, the original manually managed `app01` becomes redundant.

The final architecture therefore transitions from:

```text
Manually managed app01
```

to:

```text
Auto Scaling Group
        │
        ▼
ASG-managed Tomcat instance
```

The final project state described in the source material has the original `app01` terminated while the application tier is represented by the ASG-managed instance.

---

# 35. Final Implementation Flow

The complete implementation can be compressed into one dependency chain:

```text
1. AWS Account
        │
        ▼
2. EC2 Key Pair
        │
        ▼
3. Three Security Groups
        │
        ▼
4. EC2 Backend + Tomcat Instances
        │
        ▼
5. User Data Service Initialization
        │
        ▼
6. Route 53 Private DNS
        │
        ▼
7. Verify Backend Name Resolution
        │
        ▼
8. Build WAR with Maven
        │
        ▼
9. Upload WAR to S3
        │
        ▼
10. Download WAR to Tomcat
        │
        ▼
11. Deploy ROOT.war
        │
        ▼
12. Configure ALB + Target Group
        │
        ▼
13. Configure HTTPS with ACM
        │
        ▼
14. Configure Public DNS
        │
        ▼
15. Validate Complete Application
        │
        ▼
16. Create AMI
        │
        ▼
17. Create Launch Template
        │
        ▼
18. Create Auto Scaling Group
        │
        ▼
19. Attach Target Group
        │
        ▼
20. Enable ELB Health Checks
        │
        ▼
21. Configure Target Tracking
        │
        ▼
22. Validate ASG-managed Application
```

---

# 36. Reusable Engineering Patterns

Although this repository represents one specific AWS deployment, several engineering patterns were demonstrated.

## Layered Security

```text
Internet
   ↓
ALB SG
   ↓
Tomcat SG
   ↓
Backend SG
```

Each tier accepts traffic from the appropriate preceding layer.

---

## Name-Based Decoupling

```text
Application
     ↓
DNS Name
     ↓
Route 53
     ↓
Current Backend IP
```

The application is not directly coupled to the backend instance address.

---

## Build → Store → Deploy

```text
Build
 ↓
S3
 ↓
Deploy
```

The build environment and runtime environment are separated by an artifact store.

---

## Stabilize → Automate

```text
Static deployment
      ↓
Validation
      ↓
Auto Scaling
```

Automation is introduced only after the underlying system is known to work.

---

## Managed Service Substitution

The AWS implementation replaces some self-managed infrastructure components with AWS-managed equivalents.

For example:

```text
Local / self-managed Nginx
          ↓
Application Load Balancer

Local DNS / hosts mapping
          ↓
Route 53 Private DNS
```

The application architecture remains largely intact while selected infrastructure responsibilities move to managed AWS services.

---

# 37. Troubleshooting Approach

The implementation material emphasizes validating dependencies rather than changing many components simultaneously.

A useful troubleshooting order is:

```text
Infrastructure
     ↓
Security Groups
     ↓
Service Status
     ↓
DNS Resolution
     ↓
Artifact Availability
     ↓
Tomcat Deployment
     ↓
Target Group Health
     ↓
Application
     ↓
Backend Connectivity
```

Examples:

### Build failure

Check:

* Java version
* Maven version
* project directory
* `pom.xml`

### S3 upload failure

Check:

* AWS CLI authentication
* IAM user permissions
* configured credentials

### S3 download failure from EC2

Check:

* IAM role
* attached role permissions
* S3 access

### Application failure

Check:

* Route 53 hostnames
* `ROOT.war`
* Tomcat service
* WAR extraction
* application configuration

These troubleshooting relationships are documented in the source material.

---

# 38. Security and Repository Boundaries

The implementation used environment-specific values that should not be preserved in the public repository.

Examples include:

* AWS access keys
* secret keys
* SSH private keys
* passwords
* instance IDs
* private IP addresses
* account-specific identifiers
* environment-specific credentials

The public repository therefore documents the **implementation pattern**, not the original secret or environment state.

---

# 39. Implementation Boundary

This implementation demonstrates:

* EC2 infrastructure provisioning
* User Data initialization
* layered Security Groups
* Route 53 Private DNS
* Maven artifact generation
* S3 artifact storage
* AWS CLI
* IAM authentication
* Tomcat deployment
* Application Load Balancer
* Target Groups
* HTTPS / ACM
* public DNS mapping
* AMI creation
* Launch Templates
* Auto Scaling Groups
* target tracking
* ELB health checks
* end-to-end validation

It does not include:

* Terraform
* CloudFormation
* CI/CD
* Jenkins
* GitHub Actions
* Kubernetes
* Docker
* production monitoring
* automated database migration
* production data migration
* cloud-native re-architecture

Those capabilities are outside the implementation demonstrated by this project.

---

# 40. Final State

The final implementation represents:

```text
                    Internet
                       │
                    HTTPS
                       │
                       ▼
              Application Load
                  Balancer
                       │
                  Target Group
                       │
                       ▼
              Auto Scaling Group
                       │
                  Tomcat EC2
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       MySQL        Memcache      RabbitMQ
          ▲            ▲            ▲
          └────────────┼────────────┘
                       │
                Route 53 Private DNS
```

The application tier is now managed by Auto Scaling, while the backend services remain individually managed EC2 instances.

The project therefore ends with a working AWS lift-and-shift deployment in which:

```text
Infrastructure
      +
Service Discovery
      +
Artifact Deployment
      +
Load Balancing
      +
HTTPS
      +
Application Scaling
      +
Health Checks
      +
Validation
```

operate together as one system.

---

[← Back to README](../README.md) · [← Architecture](architecture.md) · [Validation →](validation.md) · [Limitations & Future Work →](limitations-and-future-work.md)
