# Validation

[← Back to README](../README.md) · [← Architecture](architecture.md) · [← Implementation](implementation.md) · [Limitations & Future Work →](limitations-and-future-work.md)

---

## 1. Validation Strategy

Validation was performed progressively rather than treating the final browser test as the only proof that the system worked.

The validation model is:

```text
Infrastructure
     ↓
Service Health
     ↓
Private DNS
     ↓
Application Deployment
     ↓
Target Group Health
     ↓
Load Balancer
     ↓
HTTPS / Public DNS
     ↓
Backend Connectivity
     ↓
End-to-End Application
     ↓
Auto Scaling
```

The project follows a **stabilize-first-then-automate** approach.

The fixed deployment was validated before introducing the Auto Scaling Group. This prevents automation from reproducing an incorrectly configured application instance.

---

# 2. Validation Layers

The final validation can be understood as six layers:

| Layer          | What is validated                                               |
| -------------- | --------------------------------------------------------------- |
| Infrastructure | EC2 instances and required AWS resources exist                  |
| Service        | Tomcat and backend services are running                         |
| DNS            | Private service names resolve correctly                         |
| Traffic        | ALB reaches healthy Tomcat targets                              |
| Application    | Application functions through the intended public path          |
| Scaling        | ASG-managed application instances remain healthy and functional |

Each layer depends on the previous layer.

---

# 3. Infrastructure Validation

Before validating application behavior, the underlying AWS resources must exist in the expected configuration.

The relevant resources are:

```text
EC2
Security Groups
Route 53 Private Hosted Zone
S3
Application Load Balancer
Target Group
ACM Certificate
AMI
Launch Template
Auto Scaling Group
```

The goal is not merely to confirm that resources exist.

The important question is:

> **Are the resources connected according to the intended architecture?**

---

# 4. EC2 Validation

The required EC2 roles are:

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

The first infrastructure-level check is that the required instances are present and in the expected running state.

However:

> An EC2 instance being `running` does not prove that the application running inside it is healthy.

This distinction becomes especially important for the Auto Scaling Group.

---

# 5. Service Health Validation

Each instance must provide the service expected from its role.

Conceptually:

```text
db01  → MySQL / MariaDB
mc01  → Memcache
rmq01 → RabbitMQ
app01 → Tomcat
```

For the application tier, the Tomcat service must be running before the Target Group can consider the instance healthy.

The service-level validation therefore establishes:

```text
EC2 Running
    ↓
Service Running
    ↓
Application Can Be Tested
```

---

# 6. Tomcat Pre-Check

Before introducing the Application Load Balancer, the application was first tested directly on the Tomcat instance.

The source material uses temporary access to:

```text
http://<app01-public-ip>:8080
```

The expected result is the vProfile login page.

This establishes that:

* Tomcat is running
* the WAR has been deployed
* the application is responding
* port `8080` is reachable

The temporary security-group rule allowing access from the user's IP is only for this pre-check and should not remain as part of the intended public architecture.

---

# 7. Private DNS Validation

Private DNS is a critical dependency because the application uses DNS names for its backend services.

The expected records include:

```text
db01.vprofile.in
mc01.vprofile.in
rmq01.vprofile.in
```

Each hostname must resolve to the private IP of the correct backend instance.

The validation relationship is:

```text
Application
     │
     ▼
Service Hostname
     │
     ▼
Route 53 Private DNS
     │
     ▼
Correct Private IP
     │
     ▼
Correct Backend Service
```

---

# 8. DNS Record Validation

The Route 53 console should first be checked to confirm that:

* the record names are correct
* the record types are correct
* the values are private IP addresses
* each hostname points to the intended service

A wrong name/IP combination can be particularly difficult to diagnose because DNS resolution may technically succeed while directing the application to the wrong backend.

The source material explicitly recommends checking both the Route 53 records and live resolution from inside the VPC.

---

# 9. Live DNS Resolution

DNS resolution should be tested from an instance inside the VPC, such as the application instance.

For example:

```bash
ping -c 4 db01.vprofile.in
```

The purpose here is primarily to observe the IP address returned by DNS.

The important validation is:

```text
db01.vprofile.in
       ↓
Expected db01 private IP
```

The same validation applies to:

```text
mc01.vprofile.in
rmq01.vprofile.in
```

The source material emphasizes that this test is about **DNS resolution**, not necessarily ICMP reachability.

---

# 10. Application Configuration Validation

The application configuration must use the exact DNS names created in Route 53.

The expected relationship is:

| Application dependency | DNS name            |
| ---------------------- | ------------------- |
| Database               | `db01.vprofile.in`  |
| Memcache               | `mc01.vprofile.in`  |
| RabbitMQ               | `rmq01.vprofile.in` |

A mismatch between the application configuration and the Route 53 records can cause runtime failures even when the application successfully builds.

Therefore:

```text
Route 53 Record
       ↕
application.properties
```

must agree exactly.

---

# 11. Application Artifact Validation

The deployment artifact is a WAR file.

The expected deployment chain is:

```text
Maven
  ↓
WAR
  ↓
S3
  ↓
Tomcat
  ↓
ROOT.war
  ↓
Running Application
```

Validation should establish that:

* the WAR was successfully built
* the artifact exists in S3
* the artifact was downloaded to the application instance
* Tomcat deployed the artifact
* the application is responding

---

# 12. Target Group Validation

After the Application Load Balancer is created, the Tomcat instance is registered with the Target Group.

The Target Group must eventually report the application instance as:

```text
healthy
```

The relationship is:

```text
ALB
 │
 ▼
Target Group
 │
 ▼
Tomcat :8080
 │
 ▼
Health Check
 │
 ▼
Healthy
```

If the target remains unhealthy, the ALB will not route normal application traffic to it.

The source material specifically identifies incorrect health-check ports or paths as common causes of unhealthy targets.

---

# 13. Target Group Health Troubleshooting

If a target is unhealthy, validate the following sequence:

```text
Target unhealthy
      │
      ├── Is Tomcat running?
      │
      ├── Is port 8080 reachable?
      │
      ├── Does App SG allow 8080 from LB SG?
      │
      ├── Is the health-check port correct?
      │
      ├── Is the health-check path correct?
      │
      └── Has enough time passed for health checks?
```

The source material notes that the Target Group may require multiple successful checks before the target becomes healthy.

---

# 14. Application Load Balancer Validation

Once the Target Group reports a healthy target, the ALB can be tested.

The basic flow is:

```text
Browser
   │
   ▼
ALB DNS
   │
   ▼
Target Group
   │
   ▼
Tomcat :8080
```

The ALB's DNS endpoint should display the vProfile application.

This validates that:

* the ALB is active
* the listener is configured
* the Target Group is connected
* the target is healthy
* traffic can reach Tomcat through the load balancer

The source material explicitly recommends testing the ALB DNS endpoint in a browser after confirming target health.

---

# 15. HTTPS Validation

The HTTPS path is:

```text
Browser
   │
   │ HTTPS :443
   ▼
ALB
   │
   │ HTTP :8080
   ▼
Tomcat
```

The HTTPS validation should establish that:

* the HTTPS listener exists
* the ACM certificate is attached
* the public application URL loads
* the browser reports a secure connection
* the certificate is valid for the intended domain

The source material describes checking the browser's secure connection indicator and verifying the certificate from the ALB listener configuration.

---

# 16. Public DNS Validation

The public application domain is mapped to the ALB DNS endpoint.

The expected relationship is:

```text
Application Domain
       ↓
Public DNS
       ↓
ALB DNS Name
       ↓
Application Load Balancer
```

After DNS propagation, the application should be reachable using the intended HTTPS domain rather than the ALB's generated hostname.

The source material describes the public DNS mapping through a CNAME at the domain registrar.

---

# 17. End-to-End Application Validation

Once the public DNS and HTTPS path are working, the application itself becomes the final integration test.

The complete request path is:

```text
User
 │
 ▼
Public DNS
 │
 ▼
HTTPS
 │
 ▼
Application Load Balancer
 │
 ▼
Target Group
 │
 ▼
Tomcat
 │
 ├──► MySQL
 │
 ├──► RabbitMQ
 │
 └──► Memcache
```

The application must function correctly through this entire chain.

The source material explicitly defines this as the end-to-end verification stage before Auto Scaling is introduced.

---

# 18. Database Connectivity Validation

The application login is used to validate database connectivity.

A successful login demonstrates that:

```text
Browser
   ↓
ALB
   ↓
Tomcat
   ↓
Database
```

is functioning sufficiently for authentication.

The source material specifically identifies successful login as evidence that the Tomcat application can reach MySQL because the application credentials are stored in the database.

### What this proves

* application can reach the database
* database credentials are usable
* application/database integration is functioning

### What it does not prove

It does not establish database high availability, replication, backup, or production-scale database performance.

---

# 19. RabbitMQ Connectivity Validation

RabbitMQ connectivity is validated through application behavior.

After successful login, the application dashboard should show the expected queue generation.

The validation chain is:

```text
Application
    ↓
RabbitMQ Connection
    ↓
Queue Generated
```

This provides application-level evidence that the message broker is reachable and being used.

The source material explicitly identifies queue generation as the RabbitMQ connectivity check.

---

# 20. Memcache Connectivity Validation

Memcache validation uses repeated retrieval of the same user.

The sequence is:

```text
First request
    ↓
Database
    ↓
Cache populated

Second request
    ↓
Memcache
    ↓
Cache hit
```

The application displays a message indicating that the first retrieval came from the database and inserted the result into the cache.

The same user is then requested again.

The second request should indicate that the data came from cache.

This demonstrates both:

* Memcache connectivity
* cache behavior

The source material explicitly describes this sequence as the Memcache validation method.

---

# 21. Complete Static-Deployment Validation

Before Auto Scaling is enabled, the following should all be true:

```text
✓ EC2 instances running
✓ Required services running
✓ Private DNS resolving correctly
✓ Application artifact deployed
✓ Tomcat responding
✓ Target Group healthy
✓ ALB routing traffic
✓ HTTPS working
✓ Public DNS working
✓ Application login working
✓ MySQL connectivity working
✓ RabbitMQ connectivity working
✓ Memcache behavior working
```

Only after these conditions are satisfied should the application tier be transitioned into the Auto Scaling Group.

This is the project's **stabilize-first-then-automate** boundary.

---

# 22. Auto Scaling Validation

After the Auto Scaling Group is created, validation changes slightly.

The goal is no longer only:

> Does the application work?

The goal becomes:

> **Does the application continue to work when the application instance is managed by the ASG?**

---

# 23. ASG Capacity Validation

The configured capacity is:

```text
Minimum  : 1
Desired  : 1
Maximum  : 4
```

The ASG console should reflect these values.

The source material specifies these exact project settings.

---

# 24. ASG Target Group Registration

Every instance launched by the Auto Scaling Group should automatically register with the existing Target Group.

The expected relationship is:

```text
ASG
 │
 ▼
New Tomcat Instance
 │
 ▼
Target Group
 │
 ▼
Health Check
 │
 ▼
Healthy
```

This proves that the ASG and ALB are integrated correctly.

The source material explicitly specifies attaching the ASG to the existing Target Group so launched instances are automatically registered.

---

# 25. ELB Health Check Validation

The Auto Scaling Group uses ELB health checks in addition to EC2 instance health.

This distinction is important:

```text
EC2 Health
    =
Is the VM running?

ELB Health
    =
Is the application responding?
```

With ELB health checks enabled:

```text
Tomcat unhealthy
       ↓
Target Group marks instance unhealthy
       ↓
ASG detects unhealthy state
       ↓
Instance can be replaced
```

Without ELB health checks, an EC2 instance could remain `running` even if Tomcat had failed.

---

# 26. Final ASG-Managed Application Validation

After the original manually managed application instance is removed from the Target Group and the ASG-managed instance is healthy, the application should be tested again.

The source material recommends confirming:

* only ASG-managed instance(s) remain in the Target Group
* targets are healthy
* the application URL loads
* login/authentication works
* ASG capacity values are correct
* expected SNS notifications were received for the configured events

The important end-to-end path is:

```text
Public Domain
     ↓
ALB
     ↓
Target Group
     ↓
ASG-managed Tomcat
     ↓
Backend Services
```

---

# 27. Application Validation After Instance Replacement

The final application test should not stop at checking that the new EC2 instance is running.

The application should again be accessed through the public URL.

Validation should confirm:

```text
ALB
 ↓
Healthy ASG-managed target
 ↓
Tomcat
 ↓
Database
 ↓
RabbitMQ
 ↓
Memcache
```

This confirms that the AMI and Launch Template produced an application instance capable of running the same workload as the originally validated instance.

---

# 28. Auto Scaling Behavior

The configured scaling policy uses:

```text
Target Tracking
Metric: CPU Utilization
Target: 50%
```

The intended behavior is:

```text
CPU rises above target
        ↓
Scale out

CPU falls below target
        ↓
Scale in
```

The ASG remains bounded by:

```text
1 ≤ instances ≤ 4
```

The source material specifies CPU target tracking at 50% and minimum/desired/maximum capacity of 1/1/4.

The project should distinguish between:

* **configured scaling policy**
* **observed scaling event**

Configuration alone does not prove that an actual scale-out or scale-in event was observed.

---

# 29. Evidence Mapping

The public repository should preserve only high-signal evidence from the completed environment.

Recommended evidence mapping:

| Evidence                       | What it proves                                   |
| ------------------------------ | ------------------------------------------------ |
| ALB / Target Group screenshot  | ALB configuration and healthy application target |
| Route 53 screenshot            | Private DNS records                              |
| HTTPS / certificate screenshot | ACM certificate and HTTPS listener               |
| Application screenshot         | Public application accessibility                 |
| Backend validation screenshot  | Application-level service connectivity           |
| Auto Scaling screenshot        | ASG configuration and capacity                   |
| ASG Target Group screenshot    | ASG-managed target registration and health       |

Evidence should be taken from the user's own completed environment.

Course screenshots should not be presented as evidence of personal execution.

---

# 30. Validation-to-Claim Mapping

The important project claims map to validation evidence as follows:

| Project Claim                        | Validation                     | Evidence                     |
| ------------------------------------ | ------------------------------ | ---------------------------- |
| AWS infrastructure deployed          | EC2/resource checks            | AWS console                  |
| Private service discovery configured | DNS resolution                 | Route 53 + resolution result |
| Application deployed                 | Tomcat/application access      | Application screenshot       |
| ALB configured                       | Target Group health            | ALB/Target Group             |
| HTTPS configured                     | Certificate + HTTPS access     | Browser + ACM                |
| Backend connectivity works           | Login / queue / cache behavior | Application evidence         |
| Application tier is ASG-managed      | ASG + Target Group             | ASG console                  |
| Application instance is healthy      | ELB health check               | Target Group                 |
| Final application works              | Public URL                     | Application screenshot       |

This keeps the repository's claims tied to concrete evidence rather than general statements.

---

# 31. Troubleshooting Validation Sequence

When validation fails, troubleshoot from the lowest dependency upward.

```text
Failure
  │
  ▼
1. EC2 state
  │
  ▼
2. Service state
  │
  ▼
3. Security Groups
  │
  ▼
4. DNS resolution
  │
  ▼
5. Application configuration
  │
  ▼
6. Tomcat
  │
  ▼
7. Target Group health
  │
  ▼
8. ALB
  │
  ▼
9. HTTPS / DNS
  │
  ▼
10. Application behavior
  │
  ▼
11. Backend integration
  │
  ▼
12. ASG behavior
```

This avoids changing multiple layers simultaneously and makes the source of a failure easier to isolate.

---

# 32. Validation Boundary

The validation performed by this project establishes that the AWS deployment can:

* run the supplied application
* expose it through an Application Load Balancer
* provide HTTPS access
* resolve backend services through private DNS
* communicate with MySQL / MariaDB
* communicate with RabbitMQ
* use Memcache
* manage the application tier through Auto Scaling
* register ASG-managed instances with the Target Group
* use ELB health checks for application-level health

It does **not** establish:

* production-scale performance
* production traffic capacity
* database high availability
* disaster recovery
* automated CI/CD validation
* full-stack high availability
* comprehensive observability
* production data migration
* zero-downtime deployment guarantees

These are outside the demonstrated validation scope.

---

# 33. Final Validation Model

The complete validation model can be compressed into:

```text
                     AWS Resources
                           │
                           ▼
                    Service Health
                           │
                           ▼
                    Private DNS
                           │
                           ▼
                   Tomcat Application
                           │
                           ▼
                     Target Group
                           │
                           ▼
                           ALB
                           │
                           ▼
                     HTTPS / DNS
                           │
                           ▼
                   Application Login
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           MySQL       RabbitMQ      Memcache
              │            │            │
              └────────────┼────────────┘
                           ▼
                 Static Deployment ✓
                           │
                           ▼
                    AMI / Launch Template
                           │
                           ▼
                    Auto Scaling Group
                           │
                           ▼
                  ASG-managed Instance
                           │
                           ▼
                    Target Group Health
                           │
                           ▼
                  Final Application Test
```

The validation strategy therefore proves the project progressively:

> **Infrastructure → connectivity → application → integration → scaling**

rather than treating a single successful browser request as proof that the entire system is correctly implemented.

---

[← Back to README](../README.md) · [← Architecture](architecture.md) · [← Implementation](implementation.md) · [Limitations & Future Work →](limitations-and-future-work.md)
