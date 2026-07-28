# Low-Level Design (LLD)

# Payment Service

**Project:** Merchant Funding & ACH Settlement Platform  
**Version:** 1.0  
**Author:** Bhavyasri  
**Date:** July 2026

---

# 1. Purpose

The Payment Service is responsible for managing the complete payment lifecycle within the Merchant Funding & ACH Settlement Platform.

It receives payment creation requests from the Funding Service, validates and persists payment information, publishes payment requests to Kafka for bank settlement processing, consumes settlement acknowledgements from the Bank Simulator, updates payment status, and notifies the Funding Service about the final payment outcome.

The Payment Service is the sole owner of payment data and settlement status. It does not maintain merchant profile information or funding request details beyond the identifiers required for payment processing.

---

# 2. Responsibilities

- Create ACH payment records
- Validate incoming payment requests
- Persist payment information
- Generate payment reference numbers
- Publish payment requests to Kafka
- Consume bank settlement acknowledgements from Kafka
- Update payment status based on settlement result
- Notify the Funding Service about final payment completion
- Retrieve payment details
- Expose internal REST APIs for payment operations

---

# 3. Package Structure

```text
payment-service
│
├── client
│     └── FundingFeignClient
│
├── config
├── controller
├── dto
├── entity
├── enums
├── exception
├── kafka
│     ├── producer
│     └── consumer
├── mapper
├── repository
├── service
│     ├── interfaces
│     └── impl
├── util
└── PaymentServiceApplication.java
```

---

# 4. Persistence Model

### Database

**MySQL**

---

### Table: payments

| Column | Type | Description |
|---------|------|-------------|
| id | UUID | Primary Key |
| payment_reference | VARCHAR(30) | Merchant-facing payment reference |
| funding_id | UUID | Associated funding request |
| merchant_id | UUID | Merchant identifier |
| settlement_account_number | VARCHAR(30) | Destination settlement account |
| amount | DECIMAL(15,2) | Payment amount |
| currency | VARCHAR(3) | Currency (Default: INR) |
| status | VARCHAR(30) | Current payment status |
| failure_reason | VARCHAR(255) | Settlement failure reason (Optional) |
| version | BIGINT | Optimistic locking version |
| created_at | TIMESTAMP | Creation timestamp |
| updated_at | TIMESTAMP | Last modified timestamp |

---

# 5. Domain Model

## Payment

| Field | Type |
|---------|------|
| id | UUID |
| paymentReference | String |
| fundingId | UUID |
| merchantId | UUID |
| settlementAccountNumber | String |
| amount | BigDecimal |
| currency | Currency |
| status | PaymentStatus |
| failureReason | String |
| version | Long |
| createdAt | LocalDateTime |
| updatedAt | LocalDateTime |

---

## PaymentStatus

```text
CREATED
PROCESSING
COMPLETED
FAILED
```

---

## Currency

```text
INR
```

Version 1 supports only the Indian Rupee (INR).

The Currency enum has been introduced to support future multi-currency expansion without changing the domain model.

---

### Status Lifecycle

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

---

# 6. DTO Design

## CreatePaymentRequest

| Field | Type |
|---------|------|
| fundingId | UUID |
| referenceNumber | String |
| merchantId | UUID |
| amount | BigDecimal |
| currency | Currency |
| settlementAccountNumber | String |

> Used internally by the Funding Service to create a payment after a funding request has been successfully validated.

---

## PaymentResponse

| Field | Type |
|---------|------|
| paymentId | UUID |
| paymentReference | String |
| status | PaymentStatus |
| message | String |

---

## BankSettlementEvent

| Field | Type |
|---------|------|
| paymentId | UUID |
| fundingId | UUID |
| merchantId | UUID |
| settlementAccountNumber | String |
| amount | BigDecimal |
| currency | Currency |

> Published by the Payment Service to Kafka for asynchronous settlement processing by the Bank Simulator.

---

## BankSettlementResult

| Field | Type |
|---------|------|
| paymentId | UUID |
| settlementStatus | PaymentStatus |
| failureReason | String |

> Consumed from Kafka after the Bank Simulator completes settlement processing.

---

## FundingStatusUpdateRequest

| Field | Type |
|---------|------|
| fundingStatus | FundingStatus |
| failureReason | String (Optional) |

> Sent internally to the Funding Service after the final payment outcome has been determined.

---

# 7. REST APIs

## Internal APIs (Service-to-Service)

### Create Payment

#### Endpoint

```http
POST /payments
```

#### Purpose

Used internally by the Funding Service to create an ACH payment after a funding request has been successfully validated.

#### Request

```json
{
  "fundingId": "uuid",
  "referenceNumber": "FUND-202607-000001",
  "merchantId": "uuid",
  "amount": 100000,
  "currency": "INR",
  "settlementAccountNumber": "ACC1001"
}
```

#### Response

```json
{
  "paymentId": "uuid",
  "paymentReference": "PAY-202607-000001",
  "status": "PROCESSING",
  "message": "Payment created successfully."
}
```

**Response Code**

```text
201 Created
```

---

### Get Payment by ID

#### Endpoint

```http
GET /payments/{id}
```

#### Purpose

Retrieves payment details for internal microservice communication.

This endpoint is intended for trusted internal services and is not exposed through the API Gateway.

**Response Code**

```text
200 OK
404 Not Found
```

---

# 8. Component Design

## PaymentController

### Responsibilities

- Handle incoming HTTP requests
- Validate request payloads
- Delegate business logic to the PaymentService
- Return standardized HTTP responses

---

## PaymentService

### Responsibilities

- Validate payment requests
- Create payment records
- Generate payment reference numbers
- Publish settlement events to Kafka
- Process settlement acknowledgements
- Update payment status
- Notify the Funding Service about payment completion
- Retrieve payment information

### Primary Methods

- createPayment()
- getPaymentById()
- publishSettlementEvent()
- processSettlementResult()
- updatePaymentStatus()
- notifyFundingService()

---

## PaymentRepository

### Responsibilities

- Persist payment records
- Retrieve payment records
- Update payment status

### Primary Methods

- save()
- findById()
- findByPaymentReference()
- existsById()

---

## FundingFeignClient

### Responsibilities

- Notify the Funding Service about the final payment outcome

### Endpoint Invoked

```http
PATCH /internal/funding/{id}/status
```

---

## PaymentProducer

### Responsibilities

- Publish BankSettlementEvent messages to Kafka
- Convert payment information into BankSettlementEvent objects
- Send events to the Bank Simulator topic

### Kafka Topic

```text
payment-settlement-request
```

---

## PaymentConsumer

### Responsibilities

- Consume BankSettlementResult messages from Kafka
- Validate settlement results
- Prevent duplicate settlement processing
- Update payment status
- Notify the Funding Service

### Kafka Topic

```text
payment-settlement-response
```

---

# 9. Exception Handling

The Payment Service uses centralized exception handling through
`@RestControllerAdvice`.

| Exception | Description | HTTP Status |
|-----------|-------------|-------------|
| PaymentNotFoundException | Payment does not exist | 404 Not Found |
| InvalidPaymentRequestException | Invalid payment request | 400 Bad Request |
| KafkaPublishException | Failed to publish settlement request | 500 Internal Server Error |
| SettlementProcessingException | Failed while processing settlement acknowledgement | 500 Internal Server Error |
| FundingServiceUnavailableException | Unable to notify Funding Service | 503 Service Unavailable |
| DuplicatePaymentException | Duplicate payment detected | 409 Conflict |
| ValidationException | Invalid request payload | 400 Bad Request |
| PaymentAlreadyCompletedException | Payment has already reached a final state | 409 Conflict |

---

# 10. Sequence Flow

## Payment Creation

```text
Funding Service
    │
    │ POST /payments
    ▼
Payment Controller
    │
    ▼
Payment Service
    │
    │ Validate Request
    ▼
Payment Repository
    │
    │ Save Payment (Status = CREATED)
    ▼
MySQL
    │
    ▼
Payment Service
    │
    │ Generate Settlement Event
    ▼
Kafka Producer
    │
    │ Publish Event
    ▼
Kafka Topic
(payment-settlement-request)
    │
    ▼
Payment Service
    │
    │ Update Status = PROCESSING
    ▼
Payment Repository
    │
    ▼
MySQL
    │
    ▼
Return PaymentResponse
```

---

## Settlement Processing

```text
Bank Simulator
    │
    │ Publish Settlement Result
    ▼
Kafka Topic
(payment-settlement-response)
    │
    ▼
Kafka Consumer
    │
    ▼
Payment Service
    │
    │ Update Payment Status
    ▼
Payment Repository
    │
    ▼
MySQL
    │
    ▼
Funding Feign Client
    │
    │ PATCH /internal/funding/{id}/status
    ▼
Funding Service
    │
    │ Update Funding Status
    ▼
Funding Database
```
---

# 11. Design Considerations

The following design decisions have been made for the Payment Service.

---

## Business Ownership

- The Payment Service owns the complete payment lifecycle.
- The Funding Service owns the funding request lifecycle.
- The Auth Service owns merchant profile information.
- Each microservice owns its own database and business data.

---

## Payment Processing

- Every funding request results in exactly one payment.
- The Payment Service persists the payment with status CREATED.
- After successfully publishing the settlement event to Kafka, the payment status is updated to PROCESSING.
- Settlement processing is performed asynchronously through Kafka.
- Payment status is updated to COMPLETED or FAILED only after receiving the final acknowledgement from the Bank Simulator.

---

## Security

- The Payment Service exposes only internal APIs.
- Payment creation requests are accepted only from trusted microservices.
- Merchant authentication is handled by the Auth Service and API Gateway.
- Internal APIs are not exposed through the API Gateway.

---

## Communication

- Payment creation uses synchronous REST communication with the Funding Service.
- Bank settlement requests are published asynchronously to Kafka.
- Settlement acknowledgements are consumed asynchronously from Kafka.
- After settlement completion, the Payment Service synchronously notifies the Funding Service through an internal REST API.

---

## Data Consistency

- One Payment belongs to exactly one Funding Request.
- The Payment Service stores the associated `fundingId` and `merchantId` as references.
- Merchant profile information is never duplicated in the Payment Service.
- Funding status is updated only after the corresponding payment reaches a final state (COMPLETED or FAILED).
- Each service owns and updates only its own business data.

---

## Event-Driven Processing

Kafka is used to decouple payment creation from bank settlement.

The Payment Service publishes settlement requests without waiting for the Bank Simulator to complete processing.

Settlement results are received asynchronously, enabling scalable and resilient payment processing.

---

## Validation

- Payment requests must reference a valid funding request.
- Settlement account number must be present before payment creation.
- Payment amount must be greater than zero.
- Currency must be supported by the platform.
- Settlement acknowledgements are validated before updating payment status.

---

## Auditing

Every payment maintains:

- createdAt
- updatedAt

These fields support auditing, reporting, operational monitoring, and troubleshooting.

---

## Reference Number Generation

Each payment is assigned a merchant-facing payment reference number.

### Format

```text
PAY-YYYYMM-XXXXXX
```

### Example

```text
PAY-202607-000001
```

Where:

- **PAY** — Payment prefix
- **YYYYMM** — Year and month of creation
- **XXXXXX** — Sequential running number

The UUID remains the internal identifier, while the payment reference number is used in APIs, logs, notifications, and operational support.

---

## Currency Management

The Payment Service currently supports only **INR**.

Using a Currency enum enables future support for additional currencies such as:

- USD
- EUR
- GBP
- AED

without requiring changes to the domain model.

---

## Concurrency Control

Optimistic locking is implemented using the `version` field.

This prevents concurrent updates from overwriting one another when multiple settlement acknowledgements, retry mechanisms, or duplicate Kafka messages attempt to modify the same payment record.

---

## Service Boundaries

The Payment Service never modifies funding request data directly.

It communicates with:

- Funding Service for payment creation requests.
- Kafka for bank settlement processing.
- Funding Service to notify the final payment outcome after settlement.

Each service owns its own business logic and persists only its own data.

---

# 12. Future Enhancements

The following enhancements are planned for future releases.

## Functional Enhancements

- Payment cancellation before settlement initiation
- Payment history filters (date, amount, status)
- Search by payment reference number
- Multi-currency support
- Partial payment support
- Bulk payment processing
- Merchant payment dashboard
- Daily payment reconciliation

---

## Technical Enhancements

- Idempotency support for payment creation
- Transactional Outbox Pattern
- Retry mechanism for Kafka publishing
- Dead Letter Queue (DLQ) for failed settlement events
- Resilience4j Circuit Breaker for Funding Service communication
- Distributed tracing using OpenTelemetry
- Audit logging
- Prometheus & Grafana monitoring
- API versioning

---

# 13. Conclusion

The Payment Service is responsible for managing the complete payment lifecycle within the Merchant Funding & ACH Settlement Platform.

It creates payment records, initiates bank settlement through Kafka, processes settlement acknowledgements, and communicates the final payment outcome back to the Funding Service.

The service follows enterprise microservice principles by maintaining clear ownership of payment data, using asynchronous messaging for settlement processing, and interacting with other services only through well-defined APIs.

The overall design provides:

- Database-per-service architecture
- Clear separation of business responsibilities
- Synchronous communication for payment creation
- Asynchronous Kafka-based bank settlement
- Reliable payment status management
- Scalable and maintainable service boundaries

The design establishes a solid foundation for future enhancements such as idempotent processing, resilience patterns, dead letter queues, transactional messaging, multi-currency support, and advanced payment capabilities while remaining simple and maintainable for the initial implementation.