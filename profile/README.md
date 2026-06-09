## Project 52 - Cloud Architecture

This repository documents the cloud architecture of Project 52, a multi-tenant intelligent document extraction platform built on AWS Serverless. It serves as a reference point for understanding the system structure, the services involved, and the end-to-end processing flow.

## Objective

Clearly describe the system architecture, its main building blocks, and the design decisions that support scalability, tenant isolation, resilience, and observability.

## Scope

This document covers the platform's high-level cloud architecture, the main request-processing pipeline, the core AWS services in use, and the operational patterns applied across the solution.

It focuses on the architectural view rather than implementation-level code details.

## Overview

Project 52 is a multi-tenant platform where companies can upload documents such as invoices, contracts, reports, and CVs, and receive structured data extracted automatically.

The platform combines **OCR with Amazon Textract** and **semantic analysis with Amazon Bedrock** in a fully asynchronous, serverless pipeline. In practical terms, it is designed not only to read text from a document, but also to interpret its meaning.

## Architecture Diagram

![Project 52 architecture diagram](./images/Diagrama_completo_DocLens_v2_1.drawio.png)

## Main Components

| Component | Description |
|-----------|-------------|
| SPA / Web client | User-facing application used to upload documents and review results. |
| Amazon CloudFront + AWS WAF | Edge layer that delivers the application and protects entry points with rate limiting and common web rules. |
| Amazon Cognito | Authentication layer that issues JWTs and carries tenant identity. |
| Amazon API Gateway | Public API entry point with JWT authorizer. |
| AWS Lambda (.NET) | Main compute layer for validation, orchestration, and document-processing steps. |
| Amazon EventBridge | Event routing layer used to decouple workflow stages. |
| AWS Step Functions | Orchestration layer for long-running and compensating flows. |
| Amazon SQS + DLQ | Queue-based asynchronous processing with retry and dead-letter handling. |
| Amazon Textract | OCR service used to extract text from uploaded documents. |
| Amazon Bedrock | Semantic analysis layer used to interpret extracted content. |
| Amazon DynamoDB | Operational data store for jobs, document metadata, and processing results. |
| Amazon S3 | Storage for uploaded source documents. |
| Amazon SNS | Notification channel for operational alerts and user-facing events. |
| Amazon CloudWatch + AWS X-Ray | Monitoring stack for logs, metrics, traces, and alarms. |

## Main Flow

1. A user uploads a document through the web application.
2. The request is authenticated through Cognito and received through API Gateway.
3. The uploaded PDF is stored in Amazon S3 and a processing job is created.
4. The event-driven workflow routes the job into the asynchronous pipeline through Amazon SQS, with orchestration support from EventBridge and Step Functions where needed.
5. A .NET Lambda function consumes the job, checks idempotency, and invokes Amazon Textract for OCR.
6. The extracted text is sent to Amazon Bedrock for semantic analysis.
7. Results and job state are stored in DynamoDB.
8. The user is notified through Amazon SNS, while logs, metrics, and traces are recorded in CloudWatch and X-Ray.

All processing is asynchronous, serverless, and designed to tolerate retries safely.

## Technologies and Services

The current architecture is based on the following stack:

- Cloud provider: **AWS**
- Compute: **AWS Lambda (.NET 8)**
- API layer: **Amazon API Gateway**
- Identity: **Amazon Cognito**
- Eventing and orchestration: **Amazon EventBridge**, **AWS Step Functions**
- Messaging: **Amazon SQS**, **Amazon SQS Dead Letter Queue**
- OCR: **Amazon Textract**
- AI analysis: **Amazon Bedrock**
- Operational database: **Amazon DynamoDB**
- File storage: **Amazon S3**
- Notifications: **Amazon SNS**
- Security: **AWS WAF**, **AWS KMS**, **AWS Secrets Manager**, **VPC Endpoints**
- Observability and monitoring: **Amazon CloudWatch**, **AWS X-Ray**
- Infrastructure as code: **AWS CDK (C#)**

## Environments

The reference architecture distinguishes the usual software lifecycle environments:

- Development: used for local validation, service integration, and early testing.
- Staging / preproduction: used to validate workflows, quotas, alarms, and resilience behavior before release.
- Production: intended for live workloads and monitored with operational alerts.

The reference topology also points to a primary AWS region in **eu-west-1** with failover considerations for **eu-west-2**.

## Deployment

Infrastructure is intended to be defined and deployed through **AWS CDK in C#**. A typical deployment flow is:

1. Prepare AWS credentials, environment variables, and tenant-aware configuration.
2. Provision or update infrastructure through AWS CDK.
3. Deploy the serverless services and event-driven workflow components.
4. Validate queue behavior, orchestration, alarms, and document-processing traces after deployment.

## Security

The current design includes the following security controls:

- **Authentication and authorization** through Amazon Cognito and JWT validation in API Gateway.
- **Multi-tenant isolation** using a logical silo model with `tenantId` propagated across tokens, messages, storage paths, and data keys.
- **S3 segregation** by tenant prefix, for example `s3://.../{tenantId}/{year}/{month}/{documentId}.pdf`.
- **DynamoDB partitioning** with tenant-aware keys such as `PK: TENANT#{tenantId}`.
- **Encryption at rest** through AWS KMS for S3 and DynamoDB-backed data.
- **Secret management** through AWS Secrets Manager.
- **Edge protection** through AWS WAF and controlled internal access through VPC endpoints.

## Observability

The platform uses the three pillars of observability:

- **Structured JSON logs** enriched with fields such as `correlationId`, `tenantId`, `documentId`, `step`, and `durationMs`.
- **Custom metrics** published through CloudWatch Embedded Metrics Format, including processing duration, processing errors, idempotency hit rate, DLQ depth, and circuit-breaker status.
- **Distributed tracing** with AWS X-Ray to follow a document across Lambda, Textract, Bedrock, and persistence layers.
- **Operational alarms** for DLQ depth, high error rate, high p99 latency, and open circuit breakers, with notifications routed through Amazon SNS.

## Architecture Decisions

The current architecture is based on the following key decisions:

- Use a fully **serverless, event-driven** architecture on AWS to scale with bursty workloads and minimize infrastructure management.
- Adopt a **multi-tenant logical silo** strategy instead of per-tenant AWS accounts or isolated databases.
- Use **idempotency with DynamoDB** to prevent duplicate processing when SQS retries messages.
- Control throughput through **Lambda reserved concurrency**, **SQS batch size**, and **visibility timeout** tuning.
- Apply resilience patterns such as **retry with backoff**, **dead-letter queues**, **circuit breaker**, and **Step Functions saga orchestration**.
- Standardize on **.NET and AWS CDK in C#** to align the application and infrastructure stack.

## Expected Outcomes

This architecture is intended to support:

- Automatic extraction of structured information from business documents.
- Strong tenant isolation and security from the start.
- Elastic scaling under bursty workloads without server management.
- A platform that is observable, resilient, and suitable for real-world operational scenarios.

## Development Roadmap

The supporting architecture notes define a four-phase roadmap:

| Month | Focus | Main services |
|-------|-------|---------------|
| 1 | Multi-tenant authentication and document upload | Cognito, API Gateway, S3 |
| 2 | OCR pipeline | Lambda .NET, SQS, Textract |
| 3 | AI analysis and resilience | Bedrock, Step Functions, DLQ |
| 4 | Results dashboard and observability | CloudWatch, X-Ray, QuickSight |

## Limitations and Next Steps

- Finalize service quotas and concurrency benchmarks for Textract and Bedrock.
- Add runbooks for DLQ replay, failure recovery, and incident response.
- Document environment-specific deployment parameters and CI/CD workflow details.
- Expand the architecture documentation with explicit ADR references and validation test cases for resilience and tenant isolation.

## Owners

List here the authors, maintainers, or people responsible for the architecture and its documentation.
