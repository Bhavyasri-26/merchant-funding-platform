# Merchant Funding & ACH Settlement Platform

> A production-ready, event-driven payment processing platform built using Java, Spring Boot, Spring Cloud, Kafka, and PostgreSQL.

---

## 📖 Project Overview

The Merchant Funding & ACH Settlement Platform is a microservices-based backend application that enables merchants to request same-day funding through secure REST APIs.

The system validates funding requests, processes merchant payments, communicates with downstream banking systems, handles asynchronous acknowledgements using Kafka, and maintains complete payment tracking and audit history.

This project is inspired by real-world banking payment processing systems while implementing original business logic and architecture.

---

## 🎯 Project Objectives

- Build a production-ready backend application
- Learn enterprise microservices architecture
- Implement asynchronous event-driven communication using Kafka
- Use Spring Cloud Gateway and Eureka for service discovery
- Apply design patterns and clean architecture
- Deploy using Docker and cloud-ready practices

---

## 🏗️ Planned Architecture

```text
                    Client
                       │
                       ▼
              Spring Cloud Gateway
                       │
                Eureka Discovery
                       │
 ┌────────────┬────────────┬────────────┬────────────┐
 │            │            │            │
 ▼            ▼            ▼            ▼
Auth      Merchant     Funding     Payment
Service    Service      Service      Service
                                         │
                                         ▼
                                      Kafka
                                         │
                                         ▼
                                   Bank Service
```

---

## 🚀 Features

### Authentication
- JWT Authentication
- Role-Based Authorization
- Refresh Tokens

### Merchant Management
- Register Merchant
- Update Merchant
- Funding Limits

### Funding
- Create Funding Request
- Duplicate Validation
- Funding Status Tracking

### Payment
- Payment Lifecycle Management
- Payment Status Tracking

### Banking
- Bank Acknowledgement Simulation
- Asynchronous Processing

### Infrastructure
- API Gateway
- Service Discovery
- Load Balancing
- Circuit Breaker
- Docker Deployment

---

## 🛠 Technology Stack

### Backend

- Java 21
- Spring Boot 3
- Spring Security
- Spring Cloud Gateway
- Spring Cloud Eureka
- Spring Cloud LoadBalancer
- OpenFeign
- Resilience4j

### Database

- PostgreSQL

### Messaging

- Apache Kafka

### DevOps

- Docker
- Docker Compose
- GitHub Actions

### Monitoring

- Spring Boot Actuator
- Prometheus
- Grafana

---

## 📂 Documentation

| Document | Description |
|----------|-------------|
| SRS | Software Requirement Specification |
| HLD | High Level Design |
| LLD | Low Level Design |
| Database Design | Database schema & ER Diagram |
| API Specification | REST API Documentation |
| Deployment Guide | Docker & Deployment |

Documentation is available inside the `/docs` directory.

---

## 📅 Development Roadmap

- [x] Project Planning
- [ ] Documentation
- [ ] Infrastructure Setup
- [ ] Authentication Service
- [ ] Merchant Service
- [ ] Funding Service
- [ ] Payment Service
- [ ] Kafka Integration
- [ ] Docker Deployment
- [ ] Monitoring
- [ ] Batch Processing

---

## 📌 Current Status

🚧 Project is under active development.

Version: **v0.1.0**
