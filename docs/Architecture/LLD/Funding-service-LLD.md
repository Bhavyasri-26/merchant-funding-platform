# Low-Level Design (LLD)

# Funding Service

**Project:** Merchant Funding & ACH Settlement Platform  
**Version:** 1.0  
**Author:** Bhavyasri  
**Date:** July 2026

---

# 1. Purpose

The Funding Service is responsible for managing merchant funding requests within the Merchant Funding & ACH Settlement Platform.

It validates incoming funding requests, persists funding data, retrieves merchant settlement account information from the Auth Service, initiates payment creation through the Payment Service, and maintains the funding request lifecycle based on payment outcomes.
The Funding Service is the sole owner of funding requests and does not maintain merchant profile information locally.

---

# 2. Responsibilities

- Create funding requests
- Validate funding request data
- Persist funding requests
- Retrieve funding requests for the authenticated merchant
- Retrieve a funding request by ID
- Retrieve merchant settlement account information from the Auth Service
- Invoke the Payment Service to create an ACH payment
- Update funding request status based on payment completion
- Expose REST APIs for funding operations

---

# 3. Package Structure

```text
funding-service
│
├── client
│     ├── AuthFeignClient
│     └── PaymentFeignClient
│
├── config
├── controller
├── dto
├── entity
├── enums
├── exception
├── mapper
├── repository
├── service
│     ├── interfaces
│     └── impl
├── util
└── FundingServiceApplication.java
```

---

# 4. Persistence Model

### Database

**MySQL**

---

### Table: funding_requests

| Column | Type | Description |
|---------|------|-------------|
| id | UUID | Primary Key |
| reference_number | VARCHAR(30) | Merchant-facing funding reference |
| merchant_id | UUID | Authenticated merchant identifier |
| amount | DECIMAL(15,2) | Requested funding amount |
| currency | VARCHAR(3) | Currency (Default: INR) |
| status | VARCHAR(30) | Funding request status |
| payment_id | UUID | Associated payment identifier |
| version | BIGINT | Optimistic locking version |
| created_at | TIMESTAMP | Creation timestamp |
| updated_at | TIMESTAMP | Last modified timestamp |

---

# 5. Domain Model

## FundingRequest

| Field | Type |
|---------|------|
| id | UUID |
| referenceNumber | String |
| merchantId | UUID |
| amount | BigDecimal |
| currency | Currency |
| status | FundingStatus |
| paymentId | UUID |
| version | Long |
| createdAt | LocalDateTime |
| updatedAt | LocalDateTime |

---

## FundingStatus

```text
CREATED
PAYMENT_INITIATED
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
PAYMENT_INITIATED
    │
    ├──────────────┐
    ▼              ▼
COMPLETED      FAILED
```

---

# 6. DTO Design

## CreateFundingRequest

| Field | Type | Validation |
|---------|------|------------|
| amount | BigDecimal | Positive value |

> The merchant identifier is extracted from the authenticated JWT token and is never supplied by the client.

---

## FundingResponse

| Field | Type |
|---------|------|
| fundingId | UUID |
| referenceNumber | String |
| paymentId | UUID |
| amount | BigDecimal |
| currency | Currency |
| status | String |
| message | String |

---

## CreatePaymentRequest

| Field | Type |
|---------|------|
| fundingId | UUID |
| referenceNumber | String |
| merchantId | UUID |
| amount | BigDecimal |
| currency | Currency |
| settlementAccountNumber | String |

> Used internally by the Funding Service while invoking the Payment Service.

---

## FundingStatusUpdateRequest

| Field | Type |
|---------|------|
| fundingStatus | FundingStatus |
| failureReason | String (Optional) |

> Used internally by the Payment Service to notify the Funding Service about the final payment outcome.

---

# 7. REST APIs

## External APIs (Merchant)

### Create Funding Request

#### Endpoint

```http
POST /funding
```

#### Request

```json
{
  "amount": 100000
}
```

#### Response

```json
{
  "fundingId": "uuid",
  "paymentId": "uuid",
  "status": "PAYMENT_INITIATED",
  "message": "Funding request created successfully."
}
```

**Response Code**

```text
201 Created
```

---

### Get Funding Request by ID

#### Endpoint

```http
GET /funding/{id}
```

#### Description

Returns the specified funding request only if it belongs to the authenticated merchant.

**Response Codes**

```text
200 OK
403 Forbidden
404 Not Found
```

---

### Get All Funding Requests

#### Endpoint

```http
GET /funding
```

#### Description

Returns all funding requests created by the authenticated merchant.

The merchant identifier is extracted from the JWT token.

**Response Code**

```text
200 OK
```

---

## Internal APIs (Service-to-Service)

### Update Funding Status

#### Endpoint

```http
PATCH /internal/funding/{id}/status
```

#### Purpose

Used internally by the Payment Service after receiving the final settlement result from the Bank Simulator.

#### Request

Success

```json
{
  "fundingStatus": "COMPLETED"
}
```

Failure

```json
{
  "fundingStatus": "FAILED",
  "failureReason": "INSUFFICIENT_BALANCE"
}
```

**Response Code**

```text
200 OK
```

---

# 8. Component Design

## FundingController

### Responsibilities

- Handle incoming HTTP requests
- Validate request payloads
- Delegate business logic to the FundingService
- Return standardized HTTP responses

---

## FundingService

### Responsibilities

- Validate funding requests
- Retrieve merchant information from the Auth Service
- Validate merchant settlement account details
- Create funding records
- Invoke the Payment Service
- Update funding status
- Retrieve funding requests

### Primary Methods

- createFunding()
- getFundingById()
- getFundingRequests()
- updateFundingStatus()

---

## FundingRepository

### Responsibilities

- Persist funding requests
- Retrieve funding requests
- Support merchant-specific queries

### Primary Methods

- save()
- findById()
- findByMerchantIdOrderByCreatedAtDesc()
- existsById()

---

## AuthFeignClient

### Responsibilities

- Retrieve merchant information from the Auth Service
- Fetch merchant settlement account details before payment creation

### Endpoint Invoked

```http
GET /internal/merchants/{merchantId}
```

### Response

```json
{
  "merchantId": "uuid",
  "merchantName": "ABC Traders Pvt Ltd",
  "settlementAccountNumber": "ACC1001",
  "role": "MERCHANT"
}
```

---

## PaymentFeignClient

### Responsibilities

- Invoke the Payment Service synchronously
- Create ACH payment
- Receive payment identifier

### Endpoint Invoked

```http
POST /payments
```

---

# 9. Exception Handling

The Funding Service uses centralized exception handling through
`@RestControllerAdvice`.

| Exception | Description | HTTP Status             |
|-----------|-------------|-------------------------|
| FundingNotFoundException | Funding request does not exist | 404 Not Found           |
| InvalidFundingAmountException | Funding amount must be greater than zero | 400 Bad Request         |
| UnauthorizedFundingAccessException | Merchant attempting to access another merchant's funding request | 403 Forbidden           |
| MerchantNotFoundException | Merchant details could not be retrieved from Auth Service | 404 Not Found           |
| AuthServiceUnavailableException | Auth Service is unavailable | 503 Service Unavailable |
| PaymentServiceUnavailableException | Payment Service is unavailable | 503 Service Unavailable |
| ValidationException | Invalid request payload | 400 Bad Request         |
| DuplicateFundingRequestException | Duplicate funding request detected | 409 Conflict            |
| MerchantInformationException | Merchant information received from Auth Service is incomplete or invalid | 500 Internal Server Error             |

---

# 10. Sequence Flow

## Funding Request Creation

```text
Merchant
    │
    │ POST /funding
    ▼
API Gateway
    │
    ▼
Funding Controller
    │
    ▼
Funding Service
    │
    │ Validate Request
    ▼
Auth Feign Client
    │
    │ GET /internal/merchants/{merchantId}
    ▼
Auth Service
    │
    │ Return MerchantResponseDto
    ▼
Funding Service
    │
    │ Validate Merchant Details
    ▼
Funding Repository
    │
    │ Save Funding Request
    ▼
MySQL
    │
    ▼
Funding Service
    │
    │ Build CreatePaymentRequest
    ▼
Payment Feign Client
    │
    │ POST /payments
    ▼
Payment Service
    │
    │ Create Payment
    ▼
Payment Database
    │
    │ Return Payment ID
    ▼
Funding Service
    │
    │ Update Funding Record
    │   • paymentId
    │   • status = PAYMENT_INITIATED
    ▼
Funding Repository
    │
    ▼
Return Response
```

---

## Funding Status Update

```text
Bank Simulator
    │
    │ Kafka Bank ACK
    ▼
Payment Service
    │
    │ Update Payment Status
    ▼
Payment Database
    │
    │ PATCH /internal/funding/{id}/status
    ▼
Funding Controller
    │
    ▼
Funding Service
    │
    │ Update Funding Status
    ▼
Funding Repository
    │
    ▼
MySQL
```

---

# 11. Design Considerations

The following design decisions have been made for the Funding Service.

---

## Business Ownership

- The Funding Service owns the complete funding request lifecycle.
- The Payment Service owns the payment lifecycle.
- The Auth Service owns merchant profile information.
- Each microservice owns its own database and business data.

---

## Merchant Information

- Merchant profile information is never stored in the Funding Service.
- Merchant settlement account details are retrieved from the Auth Service whenever a funding request is processed.
- The Funding Service stores only the `merchantId` as a reference to the merchant.
- This approach avoids data duplication and ensures the latest merchant information is always used during payment creation.

---

## Security

- Merchant identity is extracted from the authenticated JWT token.
- Clients cannot provide or modify the `merchantId`.
- Merchants can access only their own funding requests.
- Internal APIs are accessible only by trusted microservices.
- Internal APIs are not exposed through the API Gateway.

---

## Communication

- Funding creation uses synchronous REST communication with the Auth Service to retrieve merchant settlement details and with the Payment Service to initiate payment processing.
- Bank settlement processing uses asynchronous Kafka messaging.
- Payment completion is communicated by the Payment Service to the Funding Service through an internal REST endpoint after receiving the final bank acknowledgement.

---

## Data Consistency

- One Funding Request creates exactly one Payment.
- Funding Service stores only the `paymentId` as a reference.
- Payment details are owned exclusively by the Payment Service.
- Funding status is updated only after receiving confirmation from the Payment Service.
- Merchant profile information is never duplicated in the Funding Service.

---

## Validation

- Funding amount must be greater than zero.
- Funding requests cannot be created without authentication.
- Merchant details must exist in the Auth Service before payment initiation.
- Settlement account number must be available before creating a payment.
- Request validation is performed before persisting funding requests.

---

## Auditing

Every funding request maintains:

- createdAt
- updatedAt

These fields support auditing, reporting, and operational monitoring.

---

## Reference Number Generation

Each funding request is assigned a merchant-facing reference number.

### Format

```text
FUND-YYYYMM-XXXXXX
```

### Example

```text
FUND-202607-000001
```

Where:

- **FUND** — Funding request prefix
- **YYYYMM** — Year and month of creation
- **XXXXXX** — Sequential running number

The UUID remains the internal identifier, while the reference number is used in APIs, notifications, logs, and merchant communication.

---

## Currency Management

The Funding Service currently supports only **INR**.

Using a Currency enum enables future support for additional currencies such as:

- USD
- EUR
- GBP
- AED

without requiring changes to the domain model.

---

## Concurrency Control

Optimistic locking is implemented using the `version` field.

This prevents concurrent updates from overwriting one another when multiple requests attempt to modify the same funding record.

---

## Service Boundaries

The Funding Service never directly modifies payment records.

It communicates with:

- Auth Service for merchant information.
- Payment Service for payment creation.
- Payment Service notifies the Funding Service about the final payment outcome through an internal REST API.

Each service owns its own business logic and persists only its own data.

---

# 12. Future Enhancements

The following enhancements are planned for future releases.

## Functional Enhancements

- Funding cancellation before payment processing
- Funding history filters (date, amount, status)
- Pagination and sorting
- Search by funding reference number
- Merchant dashboard
- Multi-currency support
- Configurable merchant funding limits

---

## Technical Enhancements

- Idempotency support for funding creation
- Event-driven funding status updates
- Transactional Outbox Pattern
- Retry mechanism for Payment Service communication
- Retry mechanism for Auth Service communication
- Resilience4j Circuit Breaker
- Distributed tracing using OpenTelemetry
- Audit logging
- Prometheus & Grafana monitoring
- API versioning

---

# 13. Conclusion

The Funding Service is the core business service of the Merchant Funding & ACH Settlement Platform.

It is responsible for managing funding requests while delegating merchant profile management to the Auth Service and payment processing to the Payment Service.

The service follows enterprise microservice principles by maintaining clear ownership of business responsibilities, avoiding data duplication, and communicating with other services through well-defined APIs.

The overall design provides:

- Secure JWT-based access control
- Database-per-service architecture
- Clear separation of responsibilities
- Synchronous communication for funding and payment initiation
- Asynchronous processing for bank settlement
- Scalable and maintainable service boundaries

The design establishes a solid foundation for future enhancements such as event-driven communication, resilience patterns, idempotency, and multi-currency support while remaining simple and maintainable for the initial implementation.

