# User Registration Workflow

## Prerequisites

Before testing the user registration workflow, ensure the following services are **running and healthy**:

### Required Services

| Category           | Service                | Port            | Service Endpoint             | Health Endpoint                | Purpose                      |
| ------------------ | ---------------------- | --------------- | ---------------------------- | ------------------------------ | ---------------------------- |
| **Infrastructure** | RabbitMQ               | 5672 (15672 UI) | `amqp://localhost:5672`      | `http://localhost:15672`       | Event-driven messaging       |
|                    | Mailpit                | 1025 (8025 UI)  | `smtp://localhost:1025`      | `http://localhost:8025`        | Email testing                |
| **Services**       | Message Broker Service | 4000            | `http://localhost:4000/api`  | `http://localhost:4000/health` | RabbitMQ routing             |
|                    | Notification Service   | 3003            | Event consumer only          | `http://localhost:3003/health` | Email notifications          |
|                    | Audit Service          | 3004            | Event consumer only          | `http://localhost:3004/health` | Event logging & compliance   |
|                    | User Service           | 3002            | `http://localhost:3002/api`  | `http://localhost:3002/health` | User profile management      |
|                    | Auth Service           | 3001            | `http://localhost:3001/auth` | `http://localhost:3001/health` | Authentication orchestration |
| **Gateway**        | Web BFF                | 3100            | `http://localhost:3100/bff`  | `http://localhost:3100/health` | API Gateway                  |
| **Frontend**       | Web UI                 | 3000            | `http://localhost:3000`      | `http://localhost:3000`        | User interface               |

---

## Complete End-to-End Flow with Event-Driven Architecture

### **Service Client Pattern**

The Auth Service uses a **Service Client pattern** to communicate with the User Service. This pattern provides:

```
┌─────────────────────────────────────────────────────────────────────┐
│  AUTH SERVICE ARCHITECTURE                                          │
│                                                                     │
│  Controller Layer (auth.controller.js)                             │
│  ├─ Business logic & orchestration                                 │
│  ├─ Minimal validation (required fields only)                      │
│  ├─ Token generation & management                                  │
│  └─ Event publishing                                               │
│                    ↓ calls                                          │
│  Service Client Layer (userServiceClient.js)                       │
│  ├─ HTTP communication with User Service                           │
│  ├─ Request/Response transformation                                │
│  ├─ Error handling & normalization                                 │
│  ├─ Network error recovery (503 conversion)                        │
│  ├─ Response parsing (JSON/text)                                   │
│  └─ Error message extraction                                       │
│                    ↓ HTTP                                           │
│  USER SERVICE (External - Port 3002)                               │
│  ├─ User data persistence                                          │
│  ├─ Business validation (Layer 3)                                  │
│  ├─ Password hashing                                               │
│  └─ Database operations                                            │
└─────────────────────────────────────────────────────────────────────┘
```

**Benefits of Service Client Pattern:**

- ✅ **Separation of Concerns**: Controller focuses on workflow, client handles communication
- ✅ **Error Handling**: Centralized error transformation and retry logic
- ✅ **Testability**: Easy to mock service client in unit tests
- ✅ **Reusability**: Same client used by multiple controllers
- ✅ **Resilience**: Network error handling and timeout management
- ✅ **Logging**: Centralized logging of service-to-service calls

**userServiceClient.js Functions:**

- `getUserByEmail(email)` - Find user by email address
- `getUserById(id, token)` - Fetch user by ID (admin route)
- `createUser(userData)` - Create new user (registration)
- `deleteUserSelf(token)` - Self-service account deletion
- `deleteUserById(id, token)` - Admin user deletion

---

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER (Browser)                                                │
│  Fills registration form: firstName, lastName, email, phoneNumber, password, confirmPassword    │
└────────────────────────────────────┬────────────────────────────────────────────────────────────┘
                                     │
                                     │ POST /register
                                     │ Body: { firstName, lastName, email, phoneNumber, password }
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  WEB UI (React - Port 3000)                                      │
│  - Registration form validation (client-side)                                                   │
│  - Password strength indicator                                                                  │
│  - Form submission handling                                                                     │
└────────────────────────────────────┬────────────────────────────────────────────────────────────┘
                                     │
                                     │ HTTP POST /bff/auth/register
                                     │ Content-Type: application/json
                                     │ Body: { firstName, lastName, email, phoneNumber, password }
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          WEB BFF (Backend for Frontend - Port 3100)                             │
│                                                                                                  │
│  Role: API Gateway & Request Routing                                                            │
│  ├─ Receives registration request from UI                                                       │
│  ├─ Adds correlation ID for request tracking                                                    │
│  ├─ Logs request attempt                                                                        │
│  ├─ Proxies to Auth Service (no validation)                                                     │
│  └─ Handles response and error formatting                                                       │
└────────────────────────────────────┬────────────────────────────────────────────────────────────┘
                                     │
                                     │ HTTP POST /auth/register
                                     │ Headers: X-Correlation-ID: <uuid>
                                     │ Body: { email, password, firstName, lastName, phoneNumber }
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                            AUTH SERVICE (Express.js - Port 3001)                                │
│                                                                                                  │
│  Role: Authentication Workflow Orchestrator                                                     │
│                                                                                                  │
│  Step 1: Minimal Validation (Required Fields Only)                                             │
│  ├─ Check required fields present: firstName, lastName, email, password                        │
│  └─ No format/business validation (User Service handles this)                                  │
│  └─ No uniqueness check (User Service handles this)                                            │
│                                                                                                  │
│  Step 2: Create User via User Service ─────────────────────────────────────────────────┐       │
│  ├─ File: src/services/userServiceClient.js                                            │       │
│  ├─ Function: createUser(userData)                                                     │       │
│  ├─ Makes HTTP POST to User Service: POST /users                                       │       │
│  ├─ Send: { firstName, lastName, email, phoneNumber, password, roles: ['customer'] }   │       │
│  ├─ Handles responses:                                                                 │       │
│  │  ├─ Success (201): Returns user object from response                                │       │
│  │  ├─ Error (409): Extracts & throws "Email already exists" error                     │       │
│  │  ├─ Error (400): Extracts & throws validation error message                         │       │
│  │  └─ Network Error: Throws 503 "User service unavailable"                            │       │
│  ├─ User Service performs all validation:                                              │       │
│  │  ├─ Email uniqueness check                                                          │       │
│  │  ├─ Format validation (email, password, names, phone)                               │       │
│  │  └─ Business rules validation                                                       │       │
│  └─ userServiceClient returns:                                                         │       │
│     ├─ Success: User object                                                            │       │
│     └─ Error: Throws error with statusCode, message, and details                       │       │
│                                                                                         │       │
│  Step 3: Token Generation (after user created)                                         │       │
│  ├─ Generate JWT token (short-lived, 15 minutes)                                       │       │
│  │  └─ Payload: { id, email, roles }                                                   │       │
│  ├─ Generate Refresh token (long-lived, 7 days)                                        │       │
│  └─ Generate CSRF token                                                                │       │
│                                                                                         │       │
│  Step 4: Event Publishing                                                              │       │
│  ├─ Publish: auth.user.registered                                                      │       │
│  │  └─ Data: { userId, email, name, firstName, lastName, registeredAt, IP,            │       │
│  │            userAgent, correlationId, timestamp }                                    │       │
│  ├─ Publish: auth.email.verification.requested                                         │       │
│  │  └─ Data: { userId, email, verificationToken, verificationUrl,                     │       │
│  │            expiresAt, correlationId, timestamp }                                    │       │
│  └─ Events published to RabbitMQ                                                       │       │
│                                                                                         │       │
│  Step 5: Response                                                                      │       │
│  └─ Return: { message, requiresVerification: true, user: { ... } }                    │       │
│                                                                                         │       │
└─────────────────────────────────────┬───────────────────────────────────────────────────┼───────┘
                                      │                                                   │
                                      │ Response: 201 Created                             │
                                      │                                                   │
                                      ▼                                                   ▼
                      ┌───────────────────────────┐                    ┌──────────────────────────────┐
                      │      WEB BFF              │                    │    USER SERVICE              │
                      │                           │                    │    (Express.js - Port 3002)  │
                      │  Sets HTTP-only cookies:  │                    │                              │
                      │  ├─ jwt (JWT token)       │                    │  Role: User Data Domain      │
                      │  ├─ refreshToken          │                    │       & Business Validation  │
                      │  └─ csrfToken             │                    │                              │
                      │                           │                    │  Step 1: Comprehensive       │
                      │  Returns to UI:           │                    │          Validation          │
                      │  {                        │                    │  ├─ Check email uniqueness   │
                      │    message: "Registration │                    │  ├─ Validate email format    │
                      │      successful",         │                    │  │  (5-100 chars, RFC)       │
                      │    requiresVerification:  │                    │  ├─ Validate password        │
                      │      true,                │                    │  │  (6-25 chars, letter +    │
                      │    user: { ... }          │                    │  │   number)                 │
                      │  }                        │                    │  ├─ Validate firstName       │
                      └───────────────────────────┘                    │  │  (2-50 chars, letters)    │
                                      │                                │  ├─ Validate lastName        │
                                      │                                │  │  (0-50 chars, letters)    │
                                      ▼                                │  ├─ Validate phoneNumber     │
                      ┌───────────────────────────┐                    │  │  (7-20 chars, 7-15        │
                      │       WEB UI              │                    │  │   digits, optional)       │
                      │                           │                    │  └─ Validate required fields │
                      │  Shows success message:   │                    │                              │
                      │  "Please check your email │                    │  Step 2: Password Hashing    │
                      │   to verify your account" │                    │  └─ Bcrypt hash (10 rounds)  │
                      │                           │                    │                              │
                      │  JWT stored in cookie     │                    │  Step 3: User Creation       │
                      │  (for authenticated       │                    │  ├─ Create user document     │
                      │   requests after          │                    │  │  in MongoDB               │
                      │   verification)           │                    │  ├─ Set isEmailVerified:     │
                      └───────────────────────────┘                    │  │  false                    │
                                                                       │  ├─ Set isActive: true       │
                                                                       │  └─ Set default role:        │
                                                                       │     ['customer']             │
                                                                       │                              │
                                                                       │  Step 4: Event Publishing    │
                                                                       │  └─ Publish: user.created    │
                                                                       │     Data: { userId, email,   │
                                                                       │            createdAt }       │
                                                                       │                              │
                                                                       │  Step 5: Response            │
                                                                       │  └─ Return: user object      │
                                                                       │     or validation errors     │
                                                                       │                              │
                                                                       └──────────────┬───────────────┘
                                                                                      │
                                                                                      │ Publishes
                                                                                      │ user.created
                                                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          RABBITMQ MESSAGE BROKER (Port 9001)                                    │
│                                                                                                  │
│  Exchange: aioutlet.events (Topic Exchange)                                                     │
│                                                                                                  │
│  Queues & Routing Keys:                                                                         │
│  ├─ Queue: audit-service-queue                                                                  │
│  │  ├─ Bindings: auth.user.registered, user.created                                            │
│  │  └─ Consumer: Audit Service                                                                  │
│  │                                                                                               │
│  └─ Queue: notification-service-queue                                                           │
│     ├─ Bindings: auth.email.verification.requested                                             │
│     └─ Consumer: Notification Service                                                           │
│                                                                                                  │
│  Events Published During Registration:                                                          │
│  1. user.created                                                                                │
│  2. auth.user.registered                                                                        │
│  3. auth.email.verification.requested                                                           │
└──────────────────────┬─────────────────────────────────┬────────────────────────────────────────┘
                       │                                 │
                       │ Consumes                        │ Consumes
                       │ (Pull/Subscribe)                │ (Pull/Subscribe)
                       ▼                                 ▼
        ┌──────────────────────────┐       ┌───────────────────────────────┐
        │   AUDIT SERVICE          │       │   NOTIFICATION SERVICE        │
        │   (TypeScript - 3004)    │       │   (TypeScript - 3003)         │
        │                          │       │                               │
        │  Role: Audit Logging     │       │  Role: Email Notifications    │
        │                          │       │                               │
        │  Consumes Events:        │       │  Consumes Events:             │
        │  ├─ user.created         │       │  └─ auth.email.verification.  │
        │  └─ auth.user.registered │       │     requested                 │
        │                          │       │                               │
        │  Actions:                │       │  Actions:                     │
        │  ├─ Store audit record   │       │  ├─ Generate email template   │
        │  │  in MongoDB            │       │  ├─ Email content:            │
        │  ├─ Log details:          │       │  │  - Welcome message        │
        │  │  - Event type          │       │  │  - Verification link      │
        │  │  - User ID             │       │  │  - Link expiration        │
        │  │  - Email               │       │  │  - Company branding       │
        │  │  - IP address          │       │  └─ Send via email service   │
        │  │  - User agent          │       │     (SMTP/SendGrid/AWS SES)  │
        │  │  - Timestamp           │       │                               │
        │  │  - Correlation ID      │       │  Email Details:               │
        │  │  - Success status      │       │  ├─ To: user.email            │
        │  └─ Create searchable     │       │  ├─ Subject: "Verify your     │
        │     audit trail           │       │  │           email"            │
        │                          │       │  ├─ Body: HTML template        │
        │  Query Capabilities:     │       │  └─ Link: https://app.com/    │
        │  ├─ User registration     │       │           verify?token=xxx    │
        │  │  history                │       │                               │
        │  ├─ Failed registrations  │       │  Logs:                        │
        │  ├─ Registration by IP    │       │  └─ Email sent confirmation   │
        │  └─ Compliance reporting  │       │                               │
        └──────────────────────────┘       └───────────────────────────────┘
                       │                                 │
                       │ Stores in                       │ Logs in
                       ▼                                 ▼
        ┌──────────────────────────┐       ┌───────────────────────────────┐
        │  MONGODB (Audit DB)      │       │  MONGODB (Notification DB)    │
        │                          │       │                               │
        │  Collection:             │       │  Collection:                  │
        │  audit_logs              │       │  email_logs                   │
        │                          │       │                               │
        │  Document Schema:        │       │  Document Schema:             │
        │  {                       │       │  {                            │
        │    eventType: String,    │       │    type: String,              │
        │    userId: ObjectId,     │       │    to: String,                │
        │    email: String,        │       │    subject: String,           │
        │    ipAddress: String,    │       │    sentAt: Date,              │
        │    userAgent: String,    │       │    status: String,            │
        │    timestamp: Date,      │       │    correlationId: String      │
        │    correlationId: String,│       │  }                            │
        │    payload: Object,      │       └───────────────────────────────┘
        │    success: Boolean      │
        │  }                       │
        │                          │
        │  Indexes:                │
        │  ├─ userId               │
        │  ├─ eventType            │
        │  ├─ timestamp            │
        │  └─ correlationId        │
        └──────────────────────────┘
```

## Detailed Step-by-Step Flow

### **Step 1: User Input**

```
User Browser
    ↓
    Enters registration details (in sequence):
    - First Name: John
    - Last Name: Doe
    - Email: user@example.com
    - Phone Number: +1234567890
    - Password: SecurePass123!
    - Confirm Password: SecurePass123!
    ↓
    Clicks "Register" button
```

### **Step 2: Frontend Processing (Web UI)**

```
React Application (Port 3000)
    ↓
    1. Client-side validation (Layer 1)
       ├─ Required fields check (firstName, lastName, email, password)
       ├─ Email format: 5-100 chars, RFC pattern
       ├─ Password: 6-25 chars, letter + number
       ├─ FirstName: 2-50 chars, letters/spaces/hyphens/apostrophes/periods
       ├─ LastName: 0-50 chars, same pattern
       ├─ PhoneNumber: 7-20 chars, 7-15 digits (optional)
       ├─ Password confirmation match
       └─ Form completeness
    ↓
    2. HTTP POST to BFF
       URL: http://localhost:3100/bff/auth/register
       Method: POST
       Headers: Content-Type: application/json
       Body: {
         "firstName": "John",
         "lastName": "Doe",
         "email": "user@example.com",
         "phoneNumber": "+1234567890",
         "password": "SecurePass123!"
       }
```

### **Step 3: API Gateway (Web BFF)**

```
Express.js BFF (Port 3100)
    ↓
    1. Request reception
       ├─ Extract fields from req.body
       └─ No validation performed (proxies to Auth Service)
    2. Add correlation ID for tracking
    3. Log registration attempt
    4. Proxy to Auth Service
       URL: http://localhost:3001/api/auth/register
       Headers:
         - Content-Type: application/json
         - X-Correlation-ID: <uuid>
       Body: { email, password, firstName, lastName, phoneNumber }
    5. Return response to client
       ├─ Success: Pass through Auth Service response
       └─ Error: Format error response with appropriate status code
```

### **Step 4: Authentication Service**

```
Auth Service (Port 3001)
    ↓
    1. MINIMAL VALIDATION PHASE (Layer 2)
       File: src/controllers/auth.controller.js - register()
       ├─ Check required fields present: firstName, lastName, email, password
       ├─ No format/pattern validation (User Service handles this)
       └─ No uniqueness check (User Service handles this)
    ↓
    2. USER CREATION PHASE
       File: src/services/userServiceClient.js - createUser()

       Service Client responsibilities:
       ├─ Makes HTTP request to User Service
       ├─ Handles network errors (converts to 503 Service Unavailable)
       ├─ Extracts error messages from various response formats
       ├─ Parses JSON/text responses
       └─ Throws structured errors with statusCode and details

       HTTP Request:
       POST http://localhost:3002/users
       Headers: Content-Type: application/json
       Body: {
         "firstName": "John",
         "lastName": "Doe",
         "email": "user@example.com",
         "phoneNumber": "+1234567890",
         "password": "SecurePass123!",  // Will be validated & hashed by User Service
         "roles": ["customer"],
         "isEmailVerified": false,
         "isActive": true
       }

       User Service performs (via userServiceClient):
       ├─ Email uniqueness check (single DB query)
       ├─ Complete format validation (Layer 3)
       ├─ Password hashing
       └─ User creation

       userServiceClient Response handling:
       ├─ Success (201): Returns created user object
       ├─ Error (409): Throws error with statusCode 409, message "A user with this email already exists"
       ├─ Error (400): Throws error with statusCode 400, extracted validation message
       └─ Network Error: Throws error with statusCode 503, message "User service unavailable"
    ↓
    3. TOKEN GENERATION PHASE
       JWT Token:
       ├─ Algorithm: HS256
       ├─ Secret: process.env.JWT_SECRET
       ├─ Payload: { id, email, roles }
       ├─ Expiration: 15 minutes
       └─ Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

       Refresh Token:
       ├─ Algorithm: HS256
       ├─ Payload: { id, type: 'refresh' }
       ├─ Expiration: 7 days
       └─ Stored in HTTP-only cookie

       CSRF Token:
       ├─ Random UUID
       ├─ Stored in cookie
       └─ Used for state-changing requests
    ↓
    4. EMAIL VERIFICATION TOKEN GENERATION
       ├─ Algorithm: HS256
       ├─ Payload: { email }
       ├─ Expiration: 24 hours
       ├─ Token: verification_token_xyz...
       └─ URL: http://localhost:4000/api/auth/email/verify?token=...
    ↓
    5. EVENT PUBLISHING PHASE

       Event 1: auth.user.registered
       ├─ Exchange: aioutlet.events
       ├─ Routing Key: auth.user.registered
       ├─ Payload: {
       │    userId: "507f1f77bcf86cd799439011",
       │    email: "user@example.com",
       │    name: "John Doe",
       │    firstName: "John",
       │    lastName: "Doe",
       │    registeredAt: "2025-10-27T10:30:00.000Z",
       │    ipAddress: "192.168.1.100",
       │    userAgent: "Mozilla/5.0...",
       │    correlationId: "uuid-v4",
       │    timestamp: "2025-10-27T10:30:00.000Z"
       │  }
       └─ Purpose: Audit trail for registration

       Event 2: auth.email.verification.requested
       ├─ Exchange: aioutlet.events
       ├─ Routing Key: auth.email.verification.requested
       ├─ Payload: {
       │    userId: "507f1f77bcf86cd799439011",
       │    email: "user@example.com",
       │    verificationToken: "verification_token_xyz",
       │    verificationUrl: "http://localhost:4000/api/auth/email/verify?token=...",
       │    expiresAt: "2025-10-28T10:30:00.000Z",
       │    correlationId: "uuid-v4",
       │    timestamp: "2025-10-27T10:30:00.000Z"
       │  }
       └─ Purpose: Trigger email notification
    ↓
    6. RESPONSE PHASE
       HTTP 201 Created
       Body: {
         "message": "Registration successful, please verify your email.",
         "requiresVerification": true,
         "user": {
           "_id": "507f1f77bcf86cd799439011",
           "email": "user@example.com",
           "firstName": "John",
           "lastName": "Doe",
           "phoneNumber": "+1234567890",
           "isEmailVerified": false,
           "roles": ["customer"]
         }
       }
```

### **Step 5: User Service (Data Persistence)**

```
User Service (Port 3002)
    ↓
    1. VALIDATION
       ├─ Validate required fields (firstName, lastName, email, password)
       ├─ Validate email format
       └─ Check email uniqueness in MongoDB
    ↓
    2. PASSWORD HASHING
       ├─ Algorithm: Bcrypt
       ├─ Salt rounds: 10
       ├─ Input: "SecurePass123!"
       └─ Output: "$2b$10$N9qo8uLOickgx2ZMRZoMye..."
    ↓
    3. USER DOCUMENT CREATION
       MongoDB Insert:
       {
         "firstName": "John",
         "lastName": "Doe",
         "email": "user@example.com",
         "phoneNumber": "+1234567890",
         "password": "$2b$10$N9qo8uLOickgx2ZMRZoMye...",
         "isEmailVerified": false,
         "isActive": true,
         "roles": ["customer"],
         "createdAt": ISODate("2025-10-27T10:30:00.000Z"),
         "updatedAt": ISODate("2025-10-27T10:30:00.000Z"),
         "_id": ObjectId("507f1f77bcf86cd799439011")
       }
    ↓
    4. EVENT PUBLISHING
       Event: user.created
       ├─ Exchange: aioutlet.events
       ├─ Routing Key: user.created
       ├─ Payload: {
       │    userId: "507f1f77bcf86cd799439011",
       │    email: "user@example.com",
       │    createdAt: "2025-10-27T10:30:00.000Z"
       │  }
       └─ Purpose: Domain event - user record created
    ↓
    5. RESPONSE
       HTTP 201 Created
       Body: { user object without password }
```

### **Step 6: BFF Cookie Management**

```
Web BFF receives Auth Service response
    ↓
    Sets HTTP-only cookies:

    1. JWT Cookie
       Name: jwt
       Value: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
       HttpOnly: true
       Secure: true (in production)
       SameSite: Strict
       Path: /
       MaxAge: 15 minutes

    2. Refresh Token Cookie
       Name: refreshToken
       Value: refresh_token_value...
       HttpOnly: true
       Secure: true (in production)
       SameSite: Strict
       Path: /
       MaxAge: 7 days

    3. CSRF Token Cookie
       Name: csrfToken
       Value: csrf_token_uuid...
       HttpOnly: false (needs to be read by JS)
       Secure: true (in production)
       SameSite: Strict
       Path: /
       MaxAge: 7 days
    ↓
    Returns response to Web UI
```

### **Step 7: Event Consumption - Audit Service**

```
Audit Service (Port 3004)
    ↓
    RabbitMQ Consumer listening on: audit-service-queue
    ↓
    Receives events:
    1. user.created
    2. auth.user.registered
    ↓
    For each event:
    1. Parse event payload
    2. Extract metadata
    3. Create audit log document
       {
         "eventType": "auth.user.registered",
         "userId": "507f1f77bcf86cd799439011",
         "email": "user@example.com",
         "ipAddress": "192.168.1.100",
         "userAgent": "Mozilla/5.0...",
         "timestamp": "2025-10-27T10:30:00.000Z",
         "correlationId": "uuid-v4",
         "payload": { ...full event data... },
         "success": true,
         "action": "USER_REGISTRATION"
       }
    4. Insert into MongoDB audit_logs collection
    5. Log confirmation
    6. Acknowledge message to RabbitMQ
```

### **Step 8: Event Consumption - Notification Service**

```
Notification Service (Port 3003)
    ↓
    RabbitMQ Consumer listening on: notification-service-queue
    ↓
    Receives event: auth.email.verification.requested
    ↓
    1. EXTRACT EVENT DATA
       ├─ email: user@example.com
       ├─ verificationUrl: http://localhost:4000/api/auth/email/verify?token=...
       ├─ expiresAt: 2025-10-28T10:30:00.000Z
       └─ correlationId: uuid-v4
    ↓
    2. GENERATE EMAIL CONTENT
       Template: email-verification.html
       Variables:
       ├─ firstName: John
       ├─ verificationLink: <url>
       ├─ expirationHours: 24
       └─ companyName: AIOutlet
    ↓
    3. EMAIL COMPOSITION
       To: user@example.com
       From: noreply@aioutlet.com
       Subject: Verify your email address
       HTML Body:
       ┌──────────────────────────────────┐
       │ Welcome to AIOutlet, John!       │
       │                                  │
       │ Please verify your email by      │
       │ clicking the button below:       │
       │                                  │
       │ [Verify Email] (button)          │
       │                                  │
       │ This link expires in 24 hours.   │
       │                                  │
       │ If you didn't create an account, │
       │ please ignore this email.        │
       └──────────────────────────────────┘
    ↓
    4. SEND EMAIL
       SMTP/Service: SendGrid/AWS SES/SMTP server
       ├─ Establish connection
       ├─ Authenticate
       ├─ Send email
       └─ Receive confirmation
    ↓
    5. LOG EMAIL SENT
       MongoDB Document:
       {
         "type": "email-verification",
         "to": "user@example.com",
         "subject": "Verify your email address",
         "sentAt": "2025-10-27T10:30:05.000Z",
         "status": "sent",
         "correlationId": "uuid-v4",
         "messageId": "email-service-msg-id"
       }
    ↓
    6. ACKNOWLEDGE MESSAGE
       Send ACK to RabbitMQ
```

### **Step 9: User Interface Response**

```
Web UI (React)
    ↓
    Receives HTTP 201 response
    ↓
    1. Store JWT in memory (from cookie)
    2. Update application state
       ├─ isAuthenticated: false (pending verification)
       ├─ user: { ...user data... }
       └─ requiresEmailVerification: true
    ↓
    3. Display success message
       ┌──────────────────────────────────────┐
       │ ✅ Registration Successful!          │
       │                                      │
       │ We've sent a verification email to:  │
       │ user@example.com                     │
       │                                      │
       │ Please check your inbox and click    │
       │ the verification link to activate    │
       │ your account.                        │
       │                                      │
       │ Didn't receive the email?            │
       │ [Resend Verification Email]          │
       └──────────────────────────────────────┘
    ↓
    4. Redirect to verification pending page
```

## Event Summary

### **Three Events Published During Registration:**

| Event                               | Publisher    | Consumer(s)          | Purpose                                        |
| ----------------------------------- | ------------ | -------------------- | ---------------------------------------------- |
| `user.created`                      | User Service | Audit Service        | Domain event: User record created in database  |
| `auth.user.registered`              | Auth Service | Audit Service        | Workflow event: Registration process completed |
| `auth.email.verification.requested` | Auth Service | Notification Service | Workflow event: Trigger verification email     |
