# Low-Level Design (LLD)

# Bank Simulator Service

**Project:** Merchant Funding & ACH Settlement Platform
**Version:** 1.0
**Author:** Bhavyasri
**Date:** July 2026

---

# 1. Purpose

The Bank Simulator Service simulates an external banking system responsible for ACH settlement processing.

It consumes settlement requests published by the Payment Service through Apache Kafka, applies settlement decision logic, and publishes the final settlement result back to Kafka.

The Bank Simulator is intentionally designed as a stateless service to mimic an external bank integration while keeping the implementation simple and maintainable.

The service does not persist business data and does not own payment information.

---

# 2. Responsibilities

* Consume settlement requests from Kafka
* Validate incoming settlement events
* Simulate ACH settlement processing
* Apply settlement approval rules
* Determine settlement result
* Publish settlement outcomes to Kafka
* Generate settlement logs
* Simulate external banking behavior

---

# 3. Package Structure

```text
bank-simulator-service
│
├── config
├── dto
├── enums
├── exception
├── kafka
│     ├── consumer
│     └── producer
├── service
│     ├── interfaces
│     └── impl
├── util
└── BankSimulatorApplication.java
```

---

# 4. Persistence Model

### Database

The Bank Simulator Service does not maintain a database.

The service is intentionally stateless because:

* It simulates an external banking system
* It does not own payment data
* It processes requests independently
* It does not require long-term storage
* Settlement decisions are generated dynamically

Payment records remain the responsibility of the Payment Service.

---

# 5. Domain Model

The Bank Simulator does not maintain entities.

It uses DTOs and enums for processing.

---

## SettlementStatus

```text
COMPLETED
FAILED
```

---

# 6. DTO Design

## BankSettlementEvent

Consumed from Kafka.

| Field                   | Type       |
| ----------------------- | ---------- |
| paymentId               | UUID       |
| fundingId               | UUID       |
| merchantId              | UUID       |
| settlementAccountNumber | String     |
| amount                  | BigDecimal |
| currency                | Currency   |

Example:

```json
{
  "paymentId": "uuid",
  "fundingId": "uuid",
  "merchantId": "uuid",
  "settlementAccountNumber": "ACC1001",
  "amount": 100000,
  "currency": "INR"
}
```

---

## BankSettlementResult

Published to Kafka.

| Field            | Type             |
| ---------------- | ---------------- |
| paymentId        | UUID             |
| settlementStatus | SettlementStatus |
| failureReason    | String           |

Example:

```json
{
  "paymentId": "uuid",
  "settlementStatus": "COMPLETED",
  "failureReason": null
}
```

Failure example:

```json
{
  "paymentId": "uuid",
  "settlementStatus": "FAILED",
  "failureReason": "Settlement amount exceeds bank approval limit."
}
```

---

## Currency

```text
INR
```

Version 1 supports only INR.

The enum is included for future extensibility.

---

# 7. Kafka Design

---

## Consumer

### Topic

```text
payment-settlement-request
```

Published By:

* Payment Service

Consumed By:

* Bank Simulator Service

Purpose:

Receives settlement requests for ACH processing.

---

## Producer

### Topic

```text
payment-settlement-response
```

Published By:

* Bank Simulator Service

Consumed By:

* Payment Service

Purpose:

Returns settlement outcomes.

---

# 8. Component Design

## SettlementConsumer

### Responsibilities

* Consume settlement requests
* Validate incoming events
* Delegate processing to SettlementService

### Kafka Topic

```text
payment-settlement-request
```

---

## SettlementService

### Responsibilities

* Process settlement requests
* Simulate ACH processing
* Apply approval rules
* Determine settlement outcome
* Build response events
* Publish settlement result

### Primary Methods

* processSettlement()
* determineSettlementStatus()
* buildSettlementResult()
* publishSettlementResult()

---

## SettlementProducer

### Responsibilities

* Publish settlement results
* Send messages to Kafka

### Kafka Topic

```text
payment-settlement-response
```

---

# 9. Exception Handling

The Bank Simulator uses centralized exception handling.

| Exception                       | Description                  |
| ------------------------------- | ---------------------------- |
| InvalidSettlementEventException | Invalid settlement request   |
| SettlementProcessingException   | Settlement processing failed |
| KafkaPublishException           | Kafka publishing failure     |
| ValidationException             | Invalid event payload        |

---

# 10. Sequence Flow

## Settlement Processing

```text
Payment Service
        │
        │ Publish Settlement Request
        ▼
Kafka Topic
(payment-settlement-request)
        │
        ▼
SettlementConsumer
        │
        ▼
SettlementService
        │
        │ Validate Event
        │
        │ Simulate ACH Processing
        │
        │ Apply Settlement Rules
        ▼
Determine Result
(COMPLETED / FAILED)
        │
        ▼
SettlementProducer
        │
        ▼
Kafka Topic
(payment-settlement-response)
        │
        ▼
Payment Service
```

---

# 11. Settlement Decision Logic

The Bank Simulator applies deterministic settlement rules.

---

## Approval Rule

```text
If amount <= 500000

        COMPLETED

Else

        FAILED
```

---

## Failure Reason

```text
Settlement amount exceeds bank approval limit.
```

---

## Examples

### Successful Settlement

```text
Amount = 100000

Result = COMPLETED
```

---

### Failed Settlement

```text
Amount = 700000

Result = FAILED
Reason = Settlement amount exceeds bank approval limit.
```

---

This rule is intentionally deterministic to make testing, demonstrations, and implementation easier.

In real banking systems, settlement decisions would depend on:

* Account balance
* Fraud checks
* AML validation
* KYC verification
* Regulatory compliance
* Risk scoring
* Transaction limits

---

# 12. Design Considerations

---

## Stateless Service

The Bank Simulator is stateless.

It does not own business data and does not maintain a database.

---

## Event-Driven Processing

Settlement processing is asynchronous using Kafka.

This provides:

* Loose coupling
* Better scalability
* Independent deployment
* Fault isolation

---

## Deterministic Settlement Logic

Simple business rules are used to ensure predictable results.

This improves:

* Testing
* Debugging
* Demonstrations

---

## Service Independence

The Bank Simulator communicates only through Kafka.

It does not call other services directly.

---

## External System Simulation

The service is designed to imitate an external banking system.

A real bank integration can replace this service in the future without affecting internal business services.

---

# 13. Future Enhancements

## Functional Enhancements

* Randomized settlement
* Multiple bank simulations
* Configurable approval rules
* Different bank profiles
* Settlement delays

---

## Technical Enhancements

* Rule Engine Integration
* Retry Processing
* Dead Letter Queue (DLQ)
* Kafka Retry Topics
* Audit Logging
* Distributed Tracing
* External Bank API Integration
* Fraud Detection
* AML Validation
* KYC Verification

---

# 14. Conclusion

The Bank Simulator Service simulates an external ACH banking system within the Merchant Funding & ACH Settlement Platform.

The service consumes settlement requests, applies deterministic approval rules, and publishes settlement outcomes asynchronously through Kafka.

Its stateless and event-driven design keeps the implementation simple while providing a realistic representation of external banking integration.

The design enables:

* Asynchronous settlement processing
* Loose coupling
* Independent deployment
* Easy testing
* Future replacement with real banking integrations

The Bank Simulator provides a practical and maintainable approach for demonstrating enterprise payment workflows while keeping implementation complexity low.
