# High-Level Design (HLD)

## Merchant Funding & ACH Settlement Platform

**Version:** 1.0
**Author:** Bhavyasri
**Date:** July 2026

---

# 1. Introduction

This document describes the High-Level Design (HLD) of the Merchant Funding & ACH Settlement Platform.

The system follows a **Microservices Architecture** to ensure scalability, maintainability, and fault isolation. It simulates a real-world merchant funding workflow where a merchant requests funding, a payment is initiated through an ACH process, and a bank asynchronously acknowledges the transaction using Kafka.

---

# 2. System Architecture

The platform consists of the following components:

* API Gateway
* Eureka Service Registry
* Auth Service
* Funding Service
* Payment Service
* Notification Service
* Bank Simulator
* Kafka Message Broker
* PostgreSQL Databases

---

# 3. Architecture Diagram

```text
                   Client (React / Postman)
                           |
                           |
                    API Gateway
                (Spring Cloud Gateway)
                           |
        -----------------------------------------
        |           |            |              |
        |           |            |              |
 Auth Service  Funding Service Payment Service Notification Service
        |           |            |              |
        -----------------------------------------
                     |
                Kafka Message Broker
                     |
               Bank Simulator
                     |
             Kafka Response Topic
                     |
              Payment Service
                     |
               PostgreSQL Databases

           Eureka Server (Service Registry)
```

---

# 4. Microservices Overview

## 4.1 Eureka Server

### Purpose

Acts as the Service Registry where all microservices register themselves.

### Responsibilities

* Service Discovery
* Health Monitoring
* Dynamic Service Registration

---

## 4.2 API Gateway

### Purpose

Acts as the single entry point for all client requests.

### Responsibilities

* Request Routing
* JWT Validation
* Authentication Filter
* Centralized Logging
* Rate Limiting (Future Enhancement)

---

## 4.3 Auth Service

### Purpose

Handles user authentication and authorization.

### Responsibilities

* User Registration
* User Login
* JWT Token Generation
* JWT Validation

### Database

**users**

---

## 4.4 Funding Service

### Purpose

Handles merchant funding requests.

### Responsibilities

* Create Funding Request
* Validate Merchant
* Calculate Funding Details
* Invoke Payment Service
* Track Funding Status

### Database

**funding_requests**

---

## 4.5 Payment Service

### Purpose

Processes ACH payment requests.

### Responsibilities

* Create Payment
* Publish Payment Event to Kafka
* Receive Bank Acknowledgement
* Update Payment Status

### Database

**payments**

---

## 4.6 Notification Service

### Purpose

Sends notifications after payment completion.

### Responsibilities

- Notify merchants when a payment is completed successfully.
- Notify merchants when a payment fails.
- Email Notification (simulated using logs)
- SMS Notification (Future)
- Push Notification (Future)

For this project, notifications will be simulated using application logs.

Example log messages:

```text
Payment Successful: Funding of ₹100000 has been processed successfully.

Payment Failed: Funding request could not be processed. Please retry or contact support.
```
---

## 4.7 Bank Simulator

### Purpose

Simulates an external banking system.

### Responsibilities

* Consume Payment Requests
* Process ACH Settlement
* Publish Payment Result

---

# 5. Database Design

Each microservice owns its own database.

| Service         | Database   |
| --------------- | ---------- |
| Auth Service    | Auth DB    |
| Funding Service | Funding DB |
| Payment Service | Payment DB |

No database is shared between services.

This ensures:

* Loose Coupling
* Independent Deployment
* Better Scalability
* Fault Isolation

---

# 6. Communication Pattern

## Synchronous Communication

REST APIs are used for communication between services.

Example:

Funding Service → Payment Service

Technology:

* Spring Cloud OpenFeign

---

## Asynchronous Communication

Kafka is used for external bank communication.

Flow:

Payment Service

↓

Kafka Topic

↓

Bank Simulator

↓

Kafka Response Topic

↓

Payment Service

---

# 7. Kafka Topics

## payment-request-topic

Publishes:

* Payment ID
* Funding ID
* Merchant ID
* Amount

---

## payment-response-topic

Publishes:

* Payment ID
* Transaction Status
* SUCCESS / FAILED

---

# 8. Payment Lifecycle

```text
CREATED
    │
    ▼
PROCESSING
    │
    ├──────────────┐
    ▼              ▼
SUCCESS        FAILED
    │              │
    ▼              ▼
Notification   Notification
```

---

# 9. End-to-End Request Flow

```text
Merchant Login

↓

Auth Service

↓

JWT Generated

↓

Merchant Requests Funding

↓

API Gateway

↓

Funding Service

↓

Funding Request Saved

↓

Funding Service invokes Payment Service

↓

Payment Created

↓

Payment Status = CREATED

↓

Kafka Payment Request Published

↓

Bank Simulator Consumes Request

↓

Processes ACH Settlement

↓

Kafka Payment Response Published

↓

Payment Service Receives Response

↓

Payment Status Updated
(SUCCESS or FAILED)

↓

Notification Service

↓

Merchant Notified
```

---

# 10. Technology Stack

| Component         | Technology            |
| ----------------- | --------------------- |
| Language          | Java 17               |
| Framework         | Spring Boot           |
| Build Tool        | Maven                 |
| Service Discovery | Eureka Server         |
| API Gateway       | Spring Cloud Gateway  |
| Security          | Spring Security + JWT |
| Messaging         | Apache Kafka          |
| Database          | PostgreSQL            |
| ORM               | Spring Data JPA       |
| API Communication | OpenFeign             |
| Containerization  | Docker                |
| Version Control   | Git & GitHub          |

---

# 11. Design Decisions

The following architectural decisions were finalized for this project:

* Microservices architecture
* One Funding Request corresponds to one Payment
* Kafka used for asynchronous bank acknowledgements
* Separate database for each microservice
* OpenFeign used for inter-service communication
* Eureka for service discovery
* API Gateway as the single entry point
* JWT-based authentication
* Payment lifecycle:

    * CREATED
    * PROCESSING
    * SUCCESS
    * FAILED

---

# 12. Future Enhancements

* Merchant Dashboard (React)
* Admin Portal
* Retry Mechanism for Failed Payments
* Distributed Tracing
* Centralized Logging (ELK Stack)
* Redis Caching
* Kubernetes Deployment
* CI/CD Pipeline
* Email Service Integration
* SMS Integration
* Payment Retry Scheduler

---

# 13. Conclusion

The Merchant Funding & ACH Settlement Platform is designed using industry-standard microservices principles. The architecture emphasizes scalability, independent deployment, asynchronous messaging, and clear separation of responsibilities. This design closely resembles enterprise financial systems and provides a strong foundation for implementing the platform in Spring Boot.
