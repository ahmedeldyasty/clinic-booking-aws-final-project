# Multi-Specialty Clinic Appointment Booking Platform on AWS

> 🌐 Languages: **English** | [العربية](README.ar.md)

Final project for the **SAA25-S1 – AWS Solutions Architect** program.
This repository documents a highly available, secure and scalable cloud architecture for an **appointment booking platform for a multi-specialty medical clinic**, used by patients and clinic staff.

![Architecture](docs/architecture.png)

> The editable diagram is available at [`diagram/aws-architecture.drawio`](diagram/aws-architecture.drawio) (open with [draw.io](https://app.diagrams.net)).

---

## Table of Contents
1. [Project Goals](#project-goals)
2. [Architecture Overview](#architecture-overview)
3. [Request Flow](#request-flow)
4. [Network Design](#network-design)
5. [AWS Services Used](#aws-services-used)
6. [Security](#security)
7. [High Availability and Scalability](#high-availability-and-scalability)
8. [Asynchronous Notifications](#asynchronous-notifications)
9. [IAM Policies](#iam-policies)
10. [Repository Structure](#repository-structure)

---

## Project Goals
- Let patients book, reschedule and cancel appointments across multiple specialties.
- Give clinic staff a secure interface to manage schedules and patient records.
- Protect sensitive medical data (encryption, private networking, least-privilege access, auditing).
- Survive the loss of an Availability Zone with no downtime.
- Scale automatically with demand and keep costs under control.

## Architecture Overview
The platform is deployed in a single AWS Region inside one VPC (`10.0.0.0/16`) spread across **three Availability Zones (A, B, C)**. Each AZ contains three subnet tiers:

| Tier | Purpose | Resources |
|------|---------|-----------|
| Public subnet | Outbound internet access only | NAT Gateway |
| Private subnet – app | Application layer | ECS Fargate tasks |
| Private subnet – data | Data layer | RDS, ElastiCache |

## Request Flow
1. **Patients and staff** reach the platform through **Route 53** (DNS).
2. Traffic passes through **AWS WAF**, which filters common web attacks and bad bots.
3. **CloudFront** serves the static front end from the **S3 website** bucket and forwards API calls to **API Gateway**.
4. **Amazon Cognito** authenticates users and authorizes API Gateway requests (JWT).
5. API Gateway reaches the private backend through a **VPC Link** to the **Application Load Balancer (ALB)**.
6. The ALB distributes traffic across **ECS Fargate** tasks in the private app subnets of the three AZs, managed by an **Auto Scaling group**.
7. The application reads/writes **Amazon RDS** (primary data) and **ElastiCache** (caching and sessions), and stores documents in **S3 Files** through the **S3 Gateway Endpoint**.
8. Notification jobs are placed on **SQS** through an **Interface Endpoint** (traffic never leaves the AWS network).

## Network Design
- **VPC CIDR:** `10.0.0.0/16`
- **3 Availability Zones**, each with a public subnet, a private app subnet and a private data subnet.
- **One NAT Gateway per AZ** in the public subnets: private workloads get outbound-only internet access, and a failure in one AZ does not affect the others.
- **Internet Gateway** provides the public path; nothing in the private subnets is directly reachable from the internet.
- **VPC Endpoints** keep AWS service traffic private:
  - **S3 Gateway Endpoint** for S3 access (no NAT cost for S3 traffic).
  - **Interface Endpoint** (PrivateLink) for SQS.

## AWS Services Used

| Category | Service | Role in the solution |
|----------|---------|----------------------|
| Edge and DNS | Route 53 | DNS and health-based routing |
| Edge and DNS | CloudFront | CDN for the front end and API |
| Edge and DNS | AWS WAF | Web application firewall |
| Front end | S3 (Website) | Hosts the static web application |
| Identity | Amazon Cognito | User sign-up, sign-in and token issuance |
| API | API Gateway | Public API entry point, throttling, authorization |
| Networking | VPC, Subnets, IGW, NAT Gateway | Network isolation and routing |
| Networking | VPC Link | Private integration API Gateway → ALB |
| Networking | Application Load Balancer | Layer 7 load balancing |
| Networking | VPC Endpoints (S3 Gateway, Interface) | Private access to AWS services |
| Compute | ECS Fargate | Serverless containers for the application |
| Compute | Auto Scaling | Scales tasks with load |
| Database | Amazon RDS (Primary + Standby) | Relational data with Multi-AZ replication |
| Cache | Amazon ElastiCache | In-memory cache in each AZ |
| Messaging | SQS | Queue for notification jobs |
| Messaging | SNS | Fan-out of notifications (SMS / push / email) |
| Messaging | Amazon SES | Email delivery (confirmations, reminders) |
| Serverless | AWS Lambda | Processes queue messages and scheduled jobs |
| Scheduling | Amazon EventBridge | Scheduled triggers (e.g. appointment reminders) |
| Storage | S3 (Files) | Medical documents, reports, attachments |
| Security | IAM | Access control (least privilege) |
| Security | AWS KMS | Encryption keys for data at rest |
| Monitoring | AWS CloudTrail | Audit log of all API activity |

## Security
- **Defense in depth:** WAF → CloudFront → API Gateway → VPC Link → ALB → private subnets.
- **Authentication and authorization:** Cognito with API Gateway authorizers.
- **Network isolation:** application and database tiers live only in private subnets; databases are in dedicated data subnets with no route to the internet.
- **Encryption at rest:** RDS, ElastiCache, S3 and SQS encrypted with **KMS**.
- **Encryption in transit:** TLS everywhere (CloudFront, API Gateway, ALB, RDS, S3).
- **Least privilege:** dedicated IAM roles per workload (see [`policies/policies.json`](policies/policies.json)).
- **Auditing:** CloudTrail records account activity for compliance and investigation.
- **Private service access:** VPC endpoints avoid sending service traffic over the public internet.

## High Availability and Scalability
- **Multi-AZ everywhere:** ECS tasks, NAT Gateways and ElastiCache nodes in 3 AZs.
- **RDS replication:** the primary instance in AZ A replicates to standbys in AZ B and AZ C, with automatic failover.
- **Auto Scaling group:** ECS Fargate scales out/in based on CPU, memory or request count.
- **Managed edge:** CloudFront, Route 53, API Gateway and S3 are regional/global managed services with built-in redundancy.
- **Decoupling:** SQS absorbs spikes so notification processing never slows down booking requests.

## Asynchronous Notifications
1. After a booking is created or changed, the application sends a message to **SQS** (through the Interface Endpoint).
2. **Lambda** consumes the queue.
3. Lambda publishes to **SNS** (SMS/push) and sends emails through **SES**.
4. **EventBridge** triggers Lambda on a schedule to send appointment reminders.

## IAM Policies
All policies are in [`policies/policies.json`](policies/policies.json). They follow the principle of least privilege and cover:

| Policy | Attached to | Purpose |
|--------|-------------|---------|
| `ClinicEcsTaskRolePolicy` | ECS task role | Access to the files bucket, SQS and KMS |
| `ClinicEcsTaskExecutionRolePolicy` | ECS task execution role | Pull images and write logs |
| `ClinicLambdaNotificationRolePolicy` | Lambda execution role | Read SQS, publish SNS, send SES emails |
| `ClinicFilesBucketPolicy` | S3 files bucket | Enforce TLS and encryption |
| `ClinicWebsiteBucketPolicy` | S3 website bucket | Allow only CloudFront to read |
| `ClinicSqsQueuePolicy` | SQS queue | Enforce TLS |
| `ClinicCloudTrailBucketPolicy` | CloudTrail log bucket | Allow CloudTrail to write logs |

Resource names are prefixed with `clinic-` so the policies match without editing.

## Repository Structure
```
.
├── README.md              # English documentation
├── README.ar.md           # Arabic documentation
├── docs/
│   └── architecture.png   # Architecture diagram (image)
├── diagram/
│   └── aws-architecture.drawio   # Editable draw.io source
└── policies/
    └── policies.json      # IAM and resource policies
```

## Author
**Ahmed** – SAA25-S1, AWS Solutions Architect program.
