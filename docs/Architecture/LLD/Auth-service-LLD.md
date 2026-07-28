# Low-Level Design (LLD)

# Auth Service

**Project:** Merchant Funding & ACH Settlement Platform  
**Version:** 1.0  
**Author:** Bhavyasri  
**Date:** July 2026

---

# 1. Purpose

The Auth Service is responsible for authentication, authorization, and merchant profile management within the Merchant Funding & ACH Settlement Platform.

It manages merchant registration, login, password encryption, JWT token generation, and merchant information. Every merchant must authenticate through this service before accessing other microservices.

The Auth Service also acts as the **source of truth** for merchant profile information, including the merchant's settlement account details used during payment processing.

---

# 2. Responsibilities

- Register new merchants
- Authenticate users
- Encrypt passwords using BCrypt
- Generate JWT tokens
- Validate JWT tokens
- Manage merchant profile information
- Maintain merchant settlement account details
- Support role-based authorization
- Provide merchant information to internal microservices

---

# 3. Package Structure

```text
auth-service
│
├── config
├── controller
├── dto
├── entity
├── exception
├── repository
├── security
├── service
│     ├── impl
│     └── interfaces
├── util
└── AuthServiceApplication.java
```

---

# 4. Persistence Model

### Database

**MySQL**

---

### Table: users

| Column | Type | Description                        |
|---------|------|------------------------------------|
| id | UUID | Primary Key                        |
| merchant_name | VARCHAR(100) | Registered business name           |
| username | VARCHAR(50) | Unique login username              |
| email | VARCHAR(100) | Unique email address               |
| password | VARCHAR(255) | BCrypt encrypted password          |
| settlement_account_number | VARCHAR(30) | Unique Merchant settlement account |
| role | ENUM | MERCHANT / ADMIN                   |
| created_at | TIMESTAMP | Creation timestamp                 |
| updated_at | TIMESTAMP | Last modified timestamp            |

---

# 5. Domain Model

## User

| Field | Type |
|---------|------|
| id | UUID |
| merchantName | String |
| username | String |
| email | String |
| password | String |
| settlementAccountNumber | String |
| role | Role |
| createdAt | LocalDateTime |
| updatedAt | LocalDateTime |

---

## Role

```text
MERCHANT
ADMIN
```

---

### Merchant Information

The Auth Service owns all merchant profile information.

Other microservices should never maintain their own copy of merchant details.

Merchant information required by other services is obtained through internal REST APIs.

---

# 6. DTO Design

## RegisterRequestDto

| Field | Type |
|---------|------|
| merchantName | String |
| username | String |
| email | String |
| password | String |
| settlementAccountNumber | String |

---

## LoginRequestDto

| Field | Type |
|---------|------|
| username | String |
| password | String |

---

## AuthResponseDto

| Field | Type |
|---------|------|
| token | String |
| username | String |
| role | String |

---

## MerchantResponseDto

| Field                   | Type |
|-------------------------|------|
| merchantId              | UUID |
| merchantName            | String |
| settlementAccountNumber | String |
| role                    | String |

> Used internally by other microservices (such as the Funding Service) to retrieve merchant information required for business processing.

---

# 7. REST APIs

## Register Merchant

### Endpoint

```http
POST /auth/register
```

### Request

```json
{
  "merchantName": "ABC Traders Pvt Ltd",
  "username": "merchant01",
  "email": "merchant01@example.com",
  "password": "Password@123",
  "settlementAccountNumber": "ACC1001"
}
```

### Response

```json
{
  "message": "Merchant registered successfully."
}
```

---

## Login

### Endpoint

```http
POST /auth/login
```

### Request

```json
{
  "username": "merchant01",
  "password": "Password@123"
}
```

### Response

```json
{
  "token": "<JWT_TOKEN>",
  "username": "merchant01",
  "role": "MERCHANT"
}
```

---

## Get Merchant Details (Internal API)

### Endpoint

```http
GET /internal/merchants/{merchantId}
```

### Purpose

Used internally by the Funding Service to retrieve the merchant's settlement account information before initiating payment creation.

### Response

```json
{
  "merchantId": "7a3f6e3b-9e12-4f9f-a0d1-987654321abc",
  "merchantName": "ABC Traders Pvt Ltd",
  "settlementAccountNumber": "ACC1001",
  "role": "MERCHANT"
}
```

> This endpoint is intended for internal microservice communication and is not exposed to external clients through the API Gateway.

---

# 8. Service Layer

## AuthService

Methods:

- registerMerchant()
- authenticateUser()
- generateJwtToken()
- validateJwtToken()
- getUserByUsername()
- getMerchantById()

---

# 9. Repository Layer

## UserRepository

Methods:

- save()
- findById()
- findByUsername()
- findByEmail()
- existsByUsername()
- existsByEmail()
- findBySettlementAccountNumber()

---

# 10. Security Components

The Auth Service uses Spring Security with JWT-based authentication.

### Components

- JwtAuthenticationFilter
- JwtTokenProvider
- SecurityConfig
- UserDetailsServiceImpl
- PasswordEncoder (BCrypt)

Passwords are never stored in plain text.

JWT is used only for authentication and authorization.

Business information such as merchant profile details and settlement account numbers is retrieved through internal service communication rather than embedded within the JWT.

---

# 11. Exception Handling

| Exception | Description |
|-----------|-------------|
| UserAlreadyExistsException | Username or email already exists |
| InvalidCredentialsException | Incorrect username or password |
| UserNotFoundException | Merchant not found |
| JwtValidationException | Invalid or expired JWT |

---

# 12. Sequence Flow

## Merchant Registration

```text
Merchant
    │
    │ POST /auth/register
    ▼
AuthController
    │
    ▼
AuthService
    │
    │ Validate Request
    ▼
BCrypt Password Encoder
    │
    │ Encrypt Password
    ▼
UserRepository
    │
    ▼
MySQL
    │
    ▼
Return Success Response
```

---

## Merchant Login

```text
Merchant
    │
    │ POST /auth/login
    ▼
AuthController
    │
    ▼
AuthService
    │
    │ Validate Credentials
    ▼
UserRepository
    │
    ▼
MySQL
    │
    ▼
Generate JWT
    │
    ▼
Return JWT Token
```

---

## Internal Merchant Details Request

```text
Funding Service
    │
    │ GET /internal/merchants/{merchantId}
    ▼
AuthController
    │
    ▼
AuthService
    │
    ▼
UserRepository
    │
    ▼
MySQL
    │
    ▼
MerchantResponseDto
```

---

# 13. Design Considerations

The following design decisions have been made for the Auth Service.

### Authentication

- JWT is used for stateless authentication.
- Passwords are encrypted using BCrypt before storage.
- JWT contains only authentication and authorization information.
- JWT is validated by downstream microservices for secured endpoints.

---

### Merchant Management

- Every registered user is assigned the **MERCHANT** role by default.
- The **ADMIN** role is reserved for future administrative capabilities.
- Merchant profile information is owned exclusively by the Auth Service.
- Each merchant is associated with a unique settlement account number used during payment processing.
- Settlement account numbers must be unique across merchants to ensure payments are routed to the correct beneficiary account.

---

### Internal Communication

- The Auth Service exposes internal REST APIs for trusted microservices.
- The Funding Service retrieves merchant settlement account details through internal communication rather than storing duplicate merchant data.
- Internal APIs are not exposed through the API Gateway.

---

### Data Ownership

The Auth Service is the single source of truth for:

- User identity
- Merchant profile information
- Settlement account number
- Authentication credentials
- User roles

Other microservices should store only the identifiers they require and obtain merchant information through internal APIs when necessary.

---

### Security

- Passwords are never stored in plain text.
- Sensitive endpoints require authentication.
- Public endpoints are limited to merchant registration and login.
- Merchant credentials are isolated from business logic.

---

### Scalability

The Auth Service is completely independent of Funding, Payment, Notification, and Bank Simulator services.

This separation allows authentication functionality to scale independently without affecting business services.

---

# 14. Future Enhancements

The following enhancements are planned for future releases.

## Functional Enhancements

- Refresh Token support
- Forgot Password functionality
- Email Verification
- Multi-Factor Authentication (MFA)
- Account Locking after multiple failed login attempts
- Password Expiry Policy
- Merchant profile management
- Settlement account update functionality

---

## Technical Enhancements

- OAuth2 integration
- Social Login (Google, GitHub)
- Redis-based token blacklist
- Login audit history
- Rate limiting
- Distributed tracing using OpenTelemetry
- Prometheus & Grafana monitoring

---

# 15. Conclusion

The Auth Service provides secure authentication, authorization, and merchant profile management for the Merchant Funding & ACH Settlement Platform.

By isolating authentication concerns into a dedicated microservice, the system maintains clear separation of responsibilities and follows enterprise microservice principles.

The service acts as the authoritative source for merchant identity and settlement account information while allowing downstream services to focus exclusively on their own business responsibilities.

The use of Spring Security, JWT, BCrypt, and MySQL provides a secure and scalable foundation that can evolve with future enhancements such as OAuth2, Multi-Factor Authentication, and advanced account management capabilities.

