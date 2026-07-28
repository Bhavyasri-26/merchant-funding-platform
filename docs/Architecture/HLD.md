# High-Level Design (HLD)

# Merchant Funding & ACH Settlement Platform

**Version:** 2.0  
**Author:** Bhavyasri  
**Date:** July 2026

---

# 1. Introduction

This document describes the High-Level Design (HLD) of the Merchant Funding & ACH Settlement Platform.

The platform follows a Microservices Architecture to provide scalability, maintainability, fault isolation, and independent deployment of business capabilities.

The system simulates a real-world merchant funding workflow where a merchant requests funding, the funding request is validated and persisted, a payment is initiated through an ACH settlement process, an external banking system asynchronously processes the settlement through Kafka, and the merchant is notified once settlement is completed.

Each microservice owns its own business logic and database while communicating with other services only through well-defined REST APIs or asynchronous Kafka events.

---

# 2. System Architecture

The platform consists of the following components:

- API Gateway
- Eureka Service Registry
- Auth Service
- Funding Service
- Payment Service
- Notification Service
- Bank Simulator Service
- Apache Kafka
- MySQL Databases

---

# 3. Architecture Diagram

```text
                           Client
                    (React / Postman)
                           │
                           │
                    API Gateway
             (Spring Cloud Gateway)
                           │
      -------------------------------------------------
      │               │               │              │
      │               │               │              │
 Auth Service   Funding Service   Payment Service   Notification Service
      │               │               │                  ▲
      │               │               │                  │
      │               │<──────────────┘                  │
      │         Funding Status Update                    │
      │                                                  │
      └──────────────────────┬───────────────────────────┘
                             │
                        Apache Kafka
                  ┌──────────┴──────────┐
                  │                     │
 payment-settlement-request   payment-notification
                  │
                  ▼
        Bank Simulator Service
                  │
                  ▼
 payment-settlement-response

      Eureka Server (Service Registry)
```

---

# 4. Microservices Overview

## 4.1 Eureka Server

### Purpose

Acts as the Service Registry where all microservices register themselves.

### Responsibilities

- Service Discovery
- Dynamic Service Registration
- Health Monitoring
- Service Lookup

---

## 4.2 API Gateway

### Purpose

Acts as the single entry point for all client requests.

### Responsibilities

- Request Routing
- JWT Validation
- Authentication Filter
- Route Protection
- Centralized Logging
- Rate Limiting (Future Enhancement)

---

## 4.3 Auth Service

### Purpose

Handles merchant authentication and authorization.

### Responsibilities

- Merchant Registration
- Merchant Login
- JWT Token Generation
- JWT Validation
- Merchant Profile Management

### Database

**Auth Database**

Table:

- merchants

---

## 4.4 Funding Service

### Purpose

Manages merchant funding requests throughout their lifecycle.

### Responsibilities

- Create Funding Request
- Validate Merchant
- Calculate Funding Eligibility
- Generate Funding Reference Number
- Invoke Payment Service
- Receive Payment Status Updates
- Update Funding Status
- Retrieve Funding Details

### Database

**Funding Database**

Table:

- funding_requests

---

## 4.5 Payment Service

### Purpose

Manages the complete payment lifecycle for merchant funding requests.

### Responsibilities

- Create Payment
- Persist Payment Information
- Generate Payment Reference Number
- Publish Settlement Requests to Kafka
- Consume Settlement Results
- Update Payment Status
- Notify Funding Service
- Publish Notification Events

### Database

**Payment Database**

Table:

- payments

---

## 4.6 Notification Service

### Purpose

Sends merchant notifications after payment settlement is completed.

The service consumes notification events from Kafka and simulates notification delivery using application logs.

### Responsibilities

- Consume Notification Events
- Notify Merchant on Successful Payment
- Notify Merchant on Failed Payment
- Simulate Email Notifications
- SMS Notification (Future)
- Push Notification (Future)

Example log messages:

```text
Payment Successful

Funding Reference : FUND-202607-000001
Amount            : ₹100000
Status            : COMPLETED
```

```text
Payment Failed

Funding Reference : FUND-202607-000001
Reason            : Bank rejected settlement.
Status            : FAILED
```

---

## 4.7 Bank Simulator Service

### Purpose

Simulates an external banking system responsible for ACH settlement.

The Bank Simulator consumes settlement requests from Kafka, performs simulated settlement processing, and publishes the settlement result back to Kafka.

### Responsibilities

- Consume Settlement Requests
- Simulate ACH Settlement
- Publish Settlement Results

The Bank Simulator does not persist any business data.

---

# 5. Database Design

Each microservice owns its own database.

| Service | Database |
|----------|----------|
| Auth Service | Auth DB |
| Funding Service | Funding DB |
| Payment Service | Payment DB |

No database is shared between microservices.

This architecture provides:

- Loose Coupling
- Independent Deployment
- Fault Isolation
- Better Scalability
- Independent Data Ownership
- Clear Service Boundaries

---

# 6. Communication Pattern

The Merchant Funding & ACH Settlement Platform uses both synchronous REST communication and asynchronous event-driven messaging.

## 6.1 Synchronous Communication

REST APIs are used where an immediate response is required.

### Funding Service → Payment Service

The Funding Service invokes the Payment Service to create a payment after successfully validating and creating a funding request.

**Technology**

- Spring Cloud OpenFeign

---

### Payment Service → Funding Service

After receiving the final settlement result from the Bank Simulator, the Payment Service notifies the Funding Service to update the funding status.

**Technology**

- Spring Cloud OpenFeign

---

## 6.2 Asynchronous Communication

Apache Kafka is used for communication with external systems and for event-driven processing.

### Settlement Processing

```text
Payment Service
        │
        │ Publish Settlement Request
        ▼
payment-settlement-request
        │
        ▼
Bank Simulator Service
        │
        │ Simulate ACH Settlement
        ▼
payment-settlement-response
        │
        ▼
Payment Service
```

---

### Notification Processing

```text
Payment Service
        │
        │ Publish Notification Event
        ▼
payment-notification
        │
        ▼
Notification Service
        │
        ▼
Merchant Notification (Simulated)
```

Using Kafka decouples services, improves scalability, and enables resilient event-driven processing.

---

# 7. Kafka Topics

The platform uses Apache Kafka for asynchronous communication.

---

## payment-settlement-request

Published By:

- Payment Service

Consumed By:

- Bank Simulator Service

Purpose:

Carries settlement requests from the Payment Service to the Bank Simulator.

Typical payload:

- Payment ID
- Funding ID
- Merchant ID
- Settlement Account Number
- Amount
- Currency

---

## payment-settlement-response

Published By:

- Bank Simulator Service

Consumed By:

- Payment Service

Purpose:

Carries the final settlement outcome after ACH processing.

Typical payload:

- Payment ID
- Settlement Status
- Failure Reason (Optional)

---

## payment-notification

Published By:

- Payment Service

Consumed By:

- Notification Service

Purpose:

Triggers merchant notifications after settlement reaches a final state.

Typical payload:

- Funding Reference
- Payment Reference
- Merchant ID
- Payment Status
- Amount
- Failure Reason (Optional)

---

# 8. Payment Lifecycle

The Payment Service owns the complete payment lifecycle.

```text
CREATED
    │
    ▼
PROCESSING
    │
    ├───────────────┐
    ▼               ▼
COMPLETED        FAILED
```

Lifecycle description:

- **CREATED** – Payment record has been persisted.
- **PROCESSING** – Settlement request has been successfully published to Kafka.
- **COMPLETED** – Bank Simulator confirms successful settlement.
- **FAILED** – Bank Simulator reports settlement failure.

Only the Payment Service is responsible for changing payment status.

---

# 9. End-to-End Request Flow

```text
Merchant Login
        │
        ▼
Auth Service
        │
        ▼
JWT Generated
        │
        ▼
Merchant Requests Funding
        │
        ▼
API Gateway
        │
        ▼
Funding Service
        │
        │ Validate Merchant
        │
        │ Calculate Funding Details
        │
        ▼
Funding Request Saved
(Status = CREATED)
        │
        ▼
Funding Service invokes Payment Service
        │
        ▼
Payment Created
(Status = CREATED)
        │
        ▼
Payment Service publishes
payment-settlement-request
        │
        ▼
Payment Status = PROCESSING
        │
        ▼
Bank Simulator Service
        │
        │ Simulates ACH Settlement
        ▼
Publishes
payment-settlement-response
        │
        ▼
Payment Service
        │
        │ Updates Payment Status
        ▼
COMPLETED / FAILED
        │
        ├───────────────┐
        │               │
        ▼               ▼
Notify Funding      Publish
Service             payment-notification
        │               │
        ▼               ▼
Funding Status     Notification Service
Updated                 │
                        ▼
                Merchant Notified
```

---

# 10. Technology Stack

| Component | Technology |
|------------|------------|
| Programming Language | Java 17 |
| Framework | Spring Boot 3 |
| Build Tool | Maven |
| Service Discovery | Netflix Eureka |
| API Gateway | Spring Cloud Gateway |
| Security | Spring Security + JWT |
| Messaging | Apache Kafka |
| Database | MySQL |
| ORM | Spring Data JPA (Hibernate) |
| Inter-Service Communication | Spring Cloud OpenFeign |
| Containerization | Docker |
| API Testing | Postman |
| Version Control | Git & GitHub |
| Documentation | Markdown |
| IDE | IntelliJ IDEA |

---

# 11. Design Decisions

The following architectural decisions have been finalized for the Merchant Funding & ACH Settlement Platform.

---

## Microservices Architecture

The platform is designed using a microservices architecture, where each service owns a single business capability and can be developed, deployed, and scaled independently.

---

## Database per Service

Each microservice owns its own database.

No database is shared between services.

Benefits include:

- Loose Coupling
- Independent Deployment
- Fault Isolation
- Independent Scaling
- Clear Data Ownership

---

## Service Responsibilities

Each service has a clearly defined responsibility.

| Service | Responsibility |
|----------|----------------|
| Auth Service | Merchant authentication and authorization |
| Funding Service | Funding request lifecycle |
| Payment Service | Payment lifecycle and settlement processing |
| Notification Service | Merchant notifications |
| Bank Simulator Service | ACH settlement simulation |

---

## One Funding Request = One Payment

Each funding request results in exactly one payment.

This simplifies business rules, ensures clear traceability, and maintains a one-to-one relationship between funding requests and payments.

---

## Payment Lifecycle

The Payment Service exclusively owns the payment lifecycle.

```text
CREATED
    │
    ▼
PROCESSING
    │
    ├──────────────┐
    ▼              ▼
COMPLETED      FAILED
```

No other service is allowed to update payment status.

---

## Funding Lifecycle

The Funding Service owns the funding request lifecycle.

Funding status is updated only after the Payment Service communicates the final payment outcome.

Typical lifecycle:

```text
CREATED
    │
    ▼
PROCESSING
    │
    ├──────────────┐
    ▼              ▼
APPROVED       REJECTED
```

---

## Event-Driven Settlement

Communication with the external banking system is asynchronous.

Instead of waiting for the bank to complete settlement, the Payment Service publishes settlement requests to Kafka.

The Bank Simulator processes the request independently and publishes the settlement result.

Benefits include:

- Loose Coupling
- Better Throughput
- Improved Scalability
- Fault Isolation
- Non-blocking Processing

---

## Event-Driven Notifications

Merchant notifications are processed asynchronously.

Once settlement reaches a final state, the Payment Service publishes a notification event to Kafka.

The Notification Service consumes the event and sends the notification independently.

Benefits include:

- Faster Payment Processing
- Independent Notification Service
- Retry Capability
- Better Scalability

---

## REST Communication

REST APIs are used only where immediate responses are required.

Examples:

- Funding Service → Payment Service
- Payment Service → Funding Service

OpenFeign is used for internal service-to-service communication.

---

## Apache Kafka

Kafka is used for asynchronous messaging.

Current topics include:

- payment-settlement-request
- payment-settlement-response
- payment-notification

Kafka provides:

- Reliable Message Delivery
- Loose Coupling
- Scalability
- Event-Driven Architecture

---

## Optimistic Locking

The Payment Service uses optimistic locking through a version field.

This prevents concurrent updates from:

- Duplicate Kafka messages
- Retry mechanisms
- Simultaneous settlement acknowledgements

---

## Security

The platform follows JWT-based authentication.

Security responsibilities are divided as follows:

- API Gateway validates incoming JWT tokens.
- Auth Service manages authentication and token generation.
- Internal microservices communicate through trusted internal APIs.
- Internal APIs are not exposed directly to external clients.

---

## Reference Numbers

Business-friendly reference numbers are generated for operational tracking.

Examples:

Funding Reference

```text
FUND-202607-000001
```

Payment Reference

```text
PAY-202607-000001
```

Internally, UUIDs remain the primary identifiers.

---

## Auditing

Business entities maintain audit fields.

Typical audit information includes:

- createdAt
- updatedAt

These fields support:

- Operational Monitoring
- Troubleshooting
- Reporting
- Auditing

---

# 12. Future Enhancements

The following enhancements are planned for future releases.

## Functional Enhancements

- Merchant Dashboard
- Admin Portal
- Payment Cancellation
- Payment Retry
- Partial Payments
- Bulk Payment Processing
- Multi-Currency Support
- Merchant Payment History
- Funding Analytics Dashboard
- Daily Settlement Reconciliation

---

## Technical Enhancements

- Transactional Outbox Pattern
- Idempotent Payment Creation
- Dead Letter Queue (DLQ)
- Kafka Retry Topics
- Resilience4j Circuit Breaker
- OpenTelemetry Distributed Tracing
- ELK Stack Centralized Logging
- Prometheus Monitoring
- Grafana Dashboards
- Redis Caching
- Kubernetes Deployment
- CI/CD Pipeline
- API Versioning
- Distributed Configuration using Spring Cloud Config

---

# 13. Conclusion

The Merchant Funding & ACH Settlement Platform is designed using enterprise-grade microservice principles with clear service ownership, independent databases, synchronous REST communication for internal business operations, and asynchronous event-driven messaging for settlement and notification processing.

The architecture separates responsibilities across dedicated microservices:

- Auth Service manages authentication and merchant identity.
- Funding Service manages funding requests.
- Payment Service owns the complete payment lifecycle.
- Bank Simulator Service simulates external ACH settlement.
- Notification Service handles merchant notifications.

Apache Kafka enables resilient and scalable asynchronous communication, while Spring Cloud components such as Eureka, Gateway, and OpenFeign provide service discovery, centralized routing, and efficient inter-service communication.

The platform establishes a strong foundation for implementing enterprise patterns such as idempotency, retry mechanisms, transactional messaging, distributed tracing, centralized monitoring, and cloud-native deployment while remaining simple, modular, and maintainable for the initial implementation.