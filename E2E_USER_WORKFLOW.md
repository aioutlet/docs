# End-to-End User Workflow

## Complete Flow: Web UI → Services → Order Fulfillment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                    USER                                          │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              WEB-UI (React)                                      │
│  - Landing page with trending products                                          │
│  - Product catalog & search                                                     │
│  - Shopping cart UI                                                             │
│  - Checkout form                                                                │
│  - Order confirmation                                                           │
│  Port: 3000                                                                     │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │ HTTP Requests
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          WEB-BFF (Backend for Frontend)                          │
│  - Aggregates data from multiple services                                       │
│  - Request/Response transformation                                              │
│  - Authentication proxy                                                         │
│  - API composition layer                                                        │
│  Port: 4000                                                                     │
└───┬──────────────┬──────────────┬──────────────┬──────────────┬────────────────┘
    │              │              │              │              │
    │              │              │              │              │
    ▼              ▼              ▼              ▼              ▼
```

## Step-by-Step Workflow

### **1. User Registration & Login**

```
┌─────────┐    POST /auth/register    ┌──────────────┐    POST /auth/register    ┌──────────────┐    POST /users    ┌──────────────┐
│ Web UI  │──────────────────────────▶│   Web BFF    │──────────────────────────▶│ Auth Service │──────────────────▶│ User Service │
│         │                            │  Port: 4000  │                           │  Port: 3001  │                   │  Port: 3002  │
│         │                            │              │                           │              │                   │              │
│         │◀──────────────────────────│  Proxies to  │◀──────────────────────────│  - JWT Token │◀──────────────────│  - MongoDB   │
└─────────┘    201 Created            │  Auth Service│    201 Created            │  - Refresh   │    User Created   │  - Bcrypt    │
               requiresVerification    │              │    requiresVerification   │  - CSRF      │                   └──────────────┘
               JWT stored in cookie    └──────────────┘                           └──────┬───────┘
                                                                                          │ Publishes:
                                                                                          │ • auth.user.registered
                                                                                          │ • auth.email.verification.requested
                                                                                          ▼
                                                                                   ┌──────────────────────────────────┐
                                                                                   │   RabbitMQ Message Broker        │
                                                                                   │   Exchange: aioutlet.events      │
                                                                                   │                                  │
                                                                                   │   Topics:                        │
                                                                                   │   • auth.user.registered         │
                                                                                   │   • auth.email.verification.*    │
                                                                                   └──────────────────────────────────┘
                                                                                          ▲                  ▲
                                                                                          │ Subscribes      │ Subscribes
                                                                                          │ Consumes        │ Consumes
                                                    ┌─────────────────────────────────────┘                  └─────────────────────────┐
                                                    │                                                                                   │
                                          ┌─────────┴───────────┐                                                           ┌──────────┴────────┐
                                          │ Notification Service│                                                           │  Audit Service    │
                                          │    Port: 3003       │                                                           │   Port: 3004      │
                                          │                     │                                                           │                   │
                                          │ Consumes:           │                                                           │ Consumes:         │
                                          │ • auth.email.       │                                                           │ • auth.user.      │
                                          │   verification.*    │                                                           │   registered      │
                                          │                     │                                                           │                   │
                                          │ Sends verification  │                                                           │ Logs:             │
                                          │      email          │                                                           │ - Registration    │
                                          └─────────────────────┘                                                           │ - User details    │
                                                                                                                            │ - IP address      │
                                                                                                                            │ - Timestamp       │
                                                                                                                            │ - Correlation ID  │
                                                                                                                            └───────────────────┘

POST /bff/auth/register
├─ BFF proxies to auth-service POST /auth/register
├─ Auth service calls User Service POST /users
├─ User Service creates user in MongoDB
│  └─ Publishes: user.created (domain event)
├─ Auth service receives user, generates tokens
├─ Auth service publishes workflow events:
│  ├─ auth.user.registered → Consumed by Audit Service
│  └─ auth.email.verification.requested → Consumed by Notification Service
├─ BFF sets HTTP-only cookies (JWT, Refresh, CSRF)
└─ Returns user profile to UI

Events Flow:
├─ user.created (published by User Service)
│  └─ Domain event: user record created in database
│
├─ auth.user.registered (published by Auth Service)
│  ├─ Consumed by: Audit Service
│  ├─ Data: userId, email, name, firstName, lastName
│  ├─ Context: IP address, user agent, correlation ID
│  └─ Purpose: Audit trail for registration workflow
│
└─ auth.email.verification.requested (published by Auth Service)
   ├─ Consumed by: Notification Service
   ├─ Data: verification token, URL, expiresAt
   └─ Purpose: Trigger verification email

POST /bff/auth/login
├─ BFF proxies to auth-service POST /auth/login
├─ Auth service validates credentials & email verification
├─ Issues JWT + Refresh Token + CSRF
├─ BFF sets secure cookies
├─ Publishes event: auth.login → Consumed by Audit Service
└─ Returns user profile

auth.login Event (published by Auth Service):
├─ Consumed by: Audit Service
├─ Data: userId, email, sessionId (refresh token)
├─ Context: IP address, user agent, timestamp
├─ Correlation ID for request tracking
└─ Purpose: Security audit & login tracking

Login Event Flow:
Auth Service publishes → RabbitMQ (auth.login) ← Audit Service consumes
```

### **2. Browse Products**

```
┌─────────┐   GET /bff/products/trending   ┌──────────────┐
│ Web UI  │──────────────────────────────▶│   Web BFF    │
│         │   (with JWT cookie)            │  Port: 4000  │
│         │                                └──────┬───────┘
│         │                                   │ Aggregates
│         │                                   │
│         │                            ┌──────▼────────┐     ┌─────────────────┐
│         │                            │Product Service│────▶│  Inventory Svc  │
│         │                            │  Port: 8003   │     │   Port: 5001    │
│         │                            │               │     │                 │
│         │◀──────────────────────────│  - MongoDB    │     │ Check stock     │
└─────────┘  Products with inventory  │  - Products   │     │ availability    │
             & review ratings          │  - Categories │     └─────────────────┘
                                      └───────┬───────┘
                                              │
                                              ▼
                                      ┌─────────────────┐
                                      │  Review Service │
                                      │   Port: 3006    │
                                      │                 │
                                      │  - MongoDB      │
                                      │  - Ratings      │
                                      │  - Reviews      │
                                      └─────────────────┘

GET /bff/products/trending
├─ BFF aggregates data from multiple services:
│  ├─ Product Service: GET /products/trending
│  ├─ Review Service: GET /reviews for each product (average ratings)
│  └─ Inventory Service: GET /inventory for stock status
├─ Combines responses into optimized format
└─ Returns products with inventory & review ratings

GET /bff/products
├─ List products with filters (department, category, price)
├─ Aggregates inventory and review data
├─ Pagination (skip, limit)
└─ Returns enriched product catalog

GET /bff/products/{id}
├─ Get product details from Product Service
├─ Get inventory status from Inventory Service
├─ Get product reviews from Review Service
├─ Aggregate all data
└─ Returns complete product information
```

### **3. Add to Cart**

```
┌─────────┐  POST /bff/cart/items     ┌──────────────┐  POST /cart/items      ┌──────────────┐
│ Web UI  │──────────────────────────▶│   Web BFF    │──────────────────────▶│ Cart Service │
│         │  {productId, quantity}     │  Port: 4000  │  {productId, quantity} │  Port: 8085  │
│         │  JWT Cookie                │              │  JWT forwarded         │              │
│         │                            │  Validates   │                        │  - Go/Gin    │
│         │                            │  JWT & proxies│                       │  - Redis     │
│         │                            └──────────────┘                        └───┬───────┬───┘
│         │                                │       │
│         │                     Validates  │       │ Checks
│         │                      Product   │       │ Stock
│         │                                ▼       ▼
│         │                         ┌──────────┐ ┌──────────┐
│         │                         │ Product  │ │Inventory │
│         │                         │ Service  │ │ Service  │
│         │◀────────────────────────└──────────┘ └──────────┘
└─────────┘   Cart with total price

POST /cart/items
├─ Validates product exists (calls product-service)
├─ Checks inventory availability (calls inventory-service)
├─ Adds item to Redis cart
├─ Calculates total price
└─ Returns updated cart

GET /bff/cart
├─ BFF proxies to Cart Service
└─ Retrieves user's current cart from Redis

PUT /bff/cart/items/{productId}
└─ Update item quantity

DELETE /bff/cart/items/{productId}
└─ Remove item from cart

Guest Cart Support:
POST /bff/guest/cart/{guestId}/items (no JWT required)
POST /bff/cart/transfer (merge guest cart to user cart after login)
```

### **4. Checkout & Place Order**

```
┌─────────┐  POST /bff/orders         ┌──────────────┐  POST /orders         ┌───────────────┐
│ Web UI  │──────────────────────────▶│   Web BFF    │──────────────────────▶│ Order Service │
│         │  {                         │  Port: 4000  │  {                    │  Port: 5000   │
│         │    customerId,             │              │    customerId,        │               │
│         │    items: [],              │  Validates   │    items: [],         │  - C#/.NET    │
│         │    shippingAddress,        │  JWT &       │    shippingAddress,   │  - SQL Server │
│         │    billingAddress,         │  proxies     │    billingAddress,    │               │
│         │    totalAmount             │              │    totalAmount        │               │
│         │  }                         └──────────────┘  }                    └───────┬───────┘
│         │  JWT Cookie                                                               │
│         │  JWT Cookie                                                               │
│         │◀──────────────────────────────────────────────────────────────────────┘
└─────────┘   Order Created                   │
              Order ID                         │ Publishes:
              Order Number                     │ • order.created
                                              ▼
                                        ┌──────────────────────────────────┐
                                        │   RabbitMQ Message Broker        │
                                        │   Exchange: aioutlet.events      │
                                        │                                  │
                                        │   Topic: order.created           │
                                        │   Payload:                       │
                                        │   • orderId, orderNumber         │
                                        │   • customerId, totalAmount      │
                                        │   • items[], addresses           │
                                        │   • timestamp, correlationId     │
                                        └──────────────────────────────────┘
                                                       ▲
                                                       │ Subscribes & Consumes
                                                       │
                                                       │
                                        ┌──────────────┴──────────────┐
                                        │  Order Processor Service    │
                                        │  (Saga Orchestrator)        │
                                        │      Port: 8080             │
                                        │                             │
                                        │  Consumes: order.created    │
                                        │  Starts: Payment Saga       │
                                        └─────────────────────────────┘

POST /bff/orders
├─ BFF validates JWT token
├─ Forwards request to Order Service
├─ Order Service validates customer ID matches JWT
├─ Validates order items
├─ Calculates totals (subtotal, tax, shipping)
├─ Generates order number (ORD-YYYYMMDD-XXXXX)
├─ Saves to SQL Server
├─ Publishes order.created event to RabbitMQ
└─ Returns order details to BFF → UI

Authorization:
├─ Customer can only create their own orders
├─ JWT validation via auth middleware
└─ Role-based access control
```

### **5. Payment Processing (Dummy)**

```
                                        ┌──────────────────────────────────┐
                                        │   Order Processor Service        │
                                        │      Port: 8080                  │
                                        │                                  │
                                        │  Consumes: order.created         │
                                        │  Starts Saga: PAYMENT_PROCESSING │
                                        └──────────────┬───────────────────┘
                                                       │
                                                       │ Publishes:
                                                       │ • payment.processing
                                                       ▼
                                        ┌──────────────────────────────────┐
                                        │   RabbitMQ Message Broker        │
                                        │   Exchange: aioutlet.events      │
                                        │                                  │
                                        │   Topic: payment.processing      │
                                        │   Payload:                       │
                                        │   • orderId, customerId          │
                                        │   • amount, currency             │
                                        │   • paymentMethod                │
                                        │   • timestamp, correlationId     │
                                        └──────────────────────────────────┘
                                                       ▲
                                                       │ Subscribes & Consumes
                                                       │
┌───────────────────────────────────────────────────┴────────────────────────────────┐
│                        Payment Service                                              │
│                          Port: 5089                                                 │
│                                                                                     │
│  1. Consumes: payment.processing event                                             │
│  2. Validates order and amount                                                     │
│  3. Processes dummy payment (always succeeds)                                      │
│  4. Saves payment record to SQL Server                                             │
│  5. Publishes event: payment.processed                                             │
└─────────────────────────────────────┬───────────────────────────────────────────────┘
                                      │
                                      │ Publishes:
                                      │ • payment.processed
                                      ▼
                         ┌──────────────────────────────────┐
                         │   RabbitMQ Message Broker        │
                         │   Exchange: aioutlet.events      │
                         │                                  │
                         │   Topic: payment.processed       │
                         │   Payload:                       │
                         │   • orderId, paymentId           │
                         │   • amount, currency             │
                         │   • status: SUCCESS              │
                         │   • timestamp, correlationId     │
                         └──────────────────────────────────┘
                                      ▲
                                      │ Subscribes & Consumes
                                      │
                         ┌────────────┴──────────────┐
                         │ Order Processor Service   │
                         │                           │
                         │ Consumes: payment.processed│
                         │ Updates Saga:             │
                         │ PAYMENT_PROCESSING        │
                         │      ↓                    │
                         │ INVENTORY_PROCESSING      │
                         └───────────────────────────┘

POST /payments
├─ Payment validation (amount, order ID)
├─ Duplicate payment check
├─ Dummy payment processing (returns success)
├─ Payment record creation
├─ Payment ID generation
└─ Returns PaymentResultDto

Payment Flow:
order.created → payment.processing → PaymentService → payment.processed
```

### **6. Inventory Reservation**

```
                         ┌──────────────────────────────────┐
                         │   Order Processor Service        │
                         │                                  │
                         │  Saga State: INVENTORY_PROCESSING│
                         └──────────────┬───────────────────┘
                                        │
                                        │ Publishes:
                                        │ • inventory.reservation
                                        ▼
                         ┌──────────────────────────────────┐
                         │   RabbitMQ Message Broker        │
                         │   Exchange: aioutlet.events      │
                         │                                  │
                         │   Topic: inventory.reservation   │
                         │   Payload:                       │
                         │   • orderId                      │
                         │   • items: [{productId, qty}]    │
                         │   • timestamp, correlationId     │
                         └──────────────────────────────────┘
                                        ▲
                                        │ Subscribes & Consumes
                                        │
┌────────────────────────────────────┴────────────────────────────────────┐
│                     Inventory Service                                    │
│                       Port: 5001                                         │
│                                                                          │
│  1. Consumes: inventory.reservation event                               │
│  2. Checks stock availability for each item                             │
│  3. Reserves inventory (decrements available quantity)                  │
│  4. Creates reservation record in Flask/PostgreSQL                      │
│  5. Publishes event: inventory.reserved (on success)                    │
│     OR inventory.failed (on insufficient stock)                         │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 │ Publishes:
                                 │ • inventory.reserved
                                 ▼
                      ┌──────────────────────────────────┐
                      │   RabbitMQ Message Broker        │
                      │   Exchange: aioutlet.events      │
                      │                                  │
                      │   Topic: inventory.reserved      │
                      │   Payload:                       │
                      │   • orderId, reservationId       │
                      │   • items reserved               │
                      │   • timestamp, correlationId     │
                      └──────────────────────────────────┘
                                 ▲
                                 │ Subscribes & Consumes
                                 │
                      ┌──────────┴──────────────┐
                      │ Order Processor Service │
                      │                         │
                      │ Consumes: inventory.reserved
                      │ Updates Saga:           │
                      │ INVENTORY_PROCESSING    │
                      │      ↓                  │
                      │ SHIPPING_PROCESSING     │
                      └─────────────────────────┘

Inventory Operations:
├─ Reserve inventory for order items
├─ Rollback on payment failure
├─ Release on order cancellation
└─ Track inventory levels
```

### **7. Shipping Preparation**

```
                      ┌──────────────────────┐
                      │ Order Processor Svc  │
                      │                      │
                      │ Publishes:           │
                      │ shipping.preparation │
                      └──────────┬───────────┘
                                 │
                                 ▼
                      ┌──────────────────────┐
                      │   RabbitMQ Queue     │
                      │ shipping.preparation │
                      └──────────┬───────────┘
                                 │
                                 ▼
                      ┌──────────────────────┐
                      │  Shipping Service    │
                      │   (Simulated)        │
                      │                      │
                      │  1. Creates shipping │
                      │     label            │
                      │  2. Generates        │
                      │     tracking number  │
                      │  3. Publishes        │
                      │     shipping.prepared│
                      └──────────┬───────────┘
                                 │
                                 ▼
                      ┌──────────────────────┐
                      │   RabbitMQ Queue     │
                      │  shipping.prepared   │
                      └──────────┬───────────┘
                                 │
                                 ▼
                      ┌──────────────────────┐
                      │ Order Processor Svc  │
                      │                      │
                      │ Updates Saga:        │
                      │ SHIPPING_PROCESSING  │
                      │      ↓               │
                      │    COMPLETED ✅      │
                      └──────────┬───────────┘
                                 │
                   Publishes     │
                order.completed  │
                                 ▼
                      ┌──────────────────────┐
                      │   RabbitMQ Queue     │
                      │   order.completed    │
                      └──────────┬───────────┘
                                 │
                    Consumed by  │
                                 ▼
                      ┌──────────────────────┐
                      │ Notification Service │
                      │    Port: 3003        │
                      │                      │
                      │  Sends order         │
                      │  confirmation email  │
                      └──────────────────────┘
```

### **8. Order Status Updates**

```
┌─────────┐  GET /bff/orders/{orderId}  ┌──────────────┐  GET /orders/{orderId}  ┌───────────────┐
│ Web UI  │────────────────────────────▶│   Web BFF    │────────────────────────▶│ Order Service │
│         │  JWT Cookie                  │  Port: 4000  │  JWT forwarded          │               │
│         │                              │              │                         │               │
│         │◀────────────────────────────│  Validates   │◀────────────────────────│  Returns:     │
└─────────┘                              │  JWT &       │                         │  - Order      │
  Order Details                          │  proxies     │                         │  - Items      │
  Status: Processing/Shipped/Delivered   └──────────────┘                         │  - Status     │
                                                                                  │  - Payment    │
                                                                                  │  - Shipping   │
                                                                                  └───────────────┘

Order Status Flow:
Pending → Processing → Payment Processed →
Inventory Reserved → Shipping Prepared →
Shipped → Delivered

Admin Operations (via BFF):
PUT /bff/orders/{id}/status
└─ Update order status (Admin only)
```

## **Complete Event Flow (Saga Pattern)**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SAGA ORCHESTRATION                               │
│                    (Order Processor Service)                             │
│                                                                          │
│  Order Created Event                                                    │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────┐                                                  │
│  │ SAGA STARTED     │                                                  │
│  │ Status: PAYMENT_ │                                                  │
│  │  PROCESSING      │                                                  │
│  └────────┬─────────┘                                                  │
│           │                                                            │
│           ▼                                                            │
│  Publish: payment.processing                                          │
│           │                                                            │
│           ▼                                                            │
│  Wait for: payment.processed ✅ OR payment.failed ❌                   │
│           │                                                            │
│           ├─ Success ──▶ INVENTORY_PROCESSING                         │
│           │              Publish: inventory.reservation               │
│           │              Wait for: inventory.reserved ✅              │
│           │                       OR inventory.failed ❌              │
│           │                       │                                   │
│           │                       ├─ Success ──▶ SHIPPING_PROCESSING │
│           │                       │              Publish: shipping... │
│           │                       │              Wait for: shipping...│
│           │                       │                       │           │
│           │                       │              Success ──▶ COMPLETED│
│           │                       │                                   │
│           │                       └─ Failure ──▶ COMPENSATING        │
│           │                                      - Release inventory   │
│           │                                      - Refund payment      │
│           │                                                           │
│           └─ Failure ──▶ COMPENSATING                                │
│                         - Refund payment                              │
│                         - Update order status to Failed               │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

Compensation Flow (if any step fails):
├─ Shipping Failed → Cancel shipping
├─ Inventory Failed → Release inventory + Refund payment
└─ Payment Failed → Update order status to Failed
```

## **Supporting Services**

### **Audit Service** (Port: 3004)

```
All Events ──▶ Audit Service ──▶ MongoDB
                    │
                    └─ Stores audit logs:
                       - Event type (auth.login, auth.user.registered, order.created, etc.)
                       - Timestamp
                       - User ID / Customer ID
                       - Correlation ID
                       - IP address & User agent
                       - Event payload (full details)
                       - Success/Failure status

Key Audit Events:
├─ Authentication Events:
│  ├─ auth.user.registered (new user signup)
│  ├─ auth.login (successful/failed login attempts)
│  ├─ auth.logout (user logout)
│  ├─ auth.password.reset.requested
│  └─ auth.password.reset.completed
│
├─ Order Events:
│  ├─ order.created
│  ├─ order.updated
│  ├─ order.cancelled
│  └─ order.completed
│
├─ Payment Events:
│  ├─ payment.processed
│  ├─ payment.failed
│  └─ payment.refunded
│
└─ Security Events:
   ├─ Unauthorized access attempts
   ├─ Permission denied
   └─ Token validation failures

Query capabilities:
├─ Find all events for a user
├─ Track order lifecycle
├─ Security audit trail
└─ Compliance reporting
```

### **Notification Service** (Port: 3003)

```
Events consumed:
├─ auth.user.registered → Welcome email
├─ auth.email.verification.requested → Verification email
├─ order.created → Order confirmation email
├─ payment.processed → Payment receipt email
├─ order.shipped → Shipping notification email
└─ order.delivered → Delivery confirmation email
```

### **Admin Service** (Port: 3010)

```
Admin Operations:
├─ Manage users
├─ Manage products
├─ View all orders
├─ Update order status
├─ View analytics
└─ System configuration
```

### **Review Service** (Port: 3006)

```
POST /reviews
├─ Create product review
├─ Rating (1-5 stars)
└─ Review text

GET /reviews/product/{productId}
└─ Get product reviews

GET /reviews/user/{userId}
└─ Get user's reviews
```

## **Infrastructure Components**

### **Message Broker** (Port: 9001)

```
RabbitMQ Exchanges & Queues:
├─ aioutlet.events (topic exchange)
│  ├─ order.created
│  ├─ order.updated
│  ├─ order.cancelled
│  ├─ order.completed
│  ├─ payment.processing
│  ├─ payment.processed
│  ├─ payment.failed
│  ├─ inventory.reservation
│  ├─ inventory.reserved
│  ├─ inventory.failed
│  ├─ shipping.preparation
│  ├─ shipping.prepared
│  └─ shipping.failed
```

### **Databases**

```
MongoDB:
├─ User Service (users, profiles)
├─ Product Service (products, categories)
├─ Review Service (reviews, ratings)
└─ Audit Service (audit logs)

PostgreSQL:
├─ Order Processor Service (saga state)
└─ Inventory Service (stock levels)

SQL Server:
├─ Order Service (orders, order items)
└─ Payment Service (payments, transactions)

Redis:
└─ Cart Service (shopping carts)
```

## **Security & Observability**

### **Authentication Flow**

```
Web UI → Web BFF → Auth Service
              │
              ├─ JWT Token (short-lived)
              ├─ Refresh Token (long-lived)
              └─ CSRF Token

All service calls include:
├─ Authorization: Bearer <JWT>
├─ X-Correlation-ID: <UUID>
└─ X-Request-ID: <UUID>
```

### **Observability**

```
Each service logs:
├─ Correlation ID (request tracking)
├─ Trace ID (distributed tracing)
├─ Span ID (service-level tracing)
├─ Timestamp
├─ Log level (INFO, WARN, ERROR)
└─ Contextual metadata

Centralized logging:
└─ All services → Console/File logs → Log aggregation
```

## **Error Handling**

### **Saga Compensation**

```
If Payment Fails:
└─ Update order status to Failed
   └─ Notify customer

If Inventory Fails:
├─ Refund payment
├─ Update order status to Failed
└─ Notify customer

If Shipping Fails:
├─ Release inventory
├─ Refund payment
├─ Update order status to Failed
└─ Notify customer
```

### **Retry Mechanism**

```
Each saga step:
├─ Max retries: 3
├─ Exponential backoff
└─ Dead letter queue for failed events
```

## **API Gateway/BFF Pattern**

The **Web BFF** acts as an aggregation layer:

- Combines data from multiple services
- Reduces number of client requests
- Transforms responses for UI needs
- Handles authentication proxy
- Provides optimized endpoints for frontend

Example BFF endpoint:

```
GET /bff/products/trending-with-reviews
├─ Calls product-service for products
├─ Calls review-service for ratings
├─ Calls inventory-service for stock
└─ Returns aggregated response
```

---

## **Technology Stack Summary**

| Service              | Language   | Framework   | Database   | Port |
| -------------------- | ---------- | ----------- | ---------- | ---- |
| Web UI               | JavaScript | React       | -          | 3000 |
| Web BFF              | JavaScript | Express     | -          | 4000 |
| Auth Service         | JavaScript | Express     | MongoDB    | 3001 |
| User Service         | JavaScript | Express     | MongoDB    | 3002 |
| Notification Service | TypeScript | Express     | MongoDB    | 3003 |
| Audit Service        | TypeScript | Express     | MongoDB    | 3004 |
| Admin Service        | JavaScript | Express     | MongoDB    | 3010 |
| Review Service       | TypeScript | Express     | MongoDB    | 3006 |
| Product Service      | Python     | FastAPI     | MongoDB    | 8003 |
| Inventory Service    | Python     | Flask       | PostgreSQL | 5001 |
| Cart Service         | Go         | Gin         | Redis      | 8085 |
| Order Service        | C#         | .NET Core   | SQL Server | 5000 |
| Payment Service      | C#         | .NET Core   | SQL Server | 5089 |
| Order Processor      | Java       | Spring Boot | PostgreSQL | 8080 |
| Message Broker       | Go         | RabbitMQ    | -          | 9001 |

---

## **Deployment Architecture**

```
                    ┌────────────────┐
                    │   Load         │
                    │   Balancer     │
                    └────────┬───────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌─────────┐    ┌─────────┐    ┌─────────┐
        │ Web UI  │    │ Web UI  │    │ Web UI  │
        │Instance1│    │Instance2│    │Instance3│
        └────┬────┘    └────┬────┘    └────┬────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Web BFF     │
                    │   Gateway     │
                    └───────┬───────┘
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
        ▼                   ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Microservices│    │ Microservices│    │ Microservices│
│   Cluster    │    │   Cluster    │    │   Cluster    │
└──────────────┘    └──────────────┘    └──────────────┘
        │                   │                    │
        └───────────────────┼────────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   RabbitMQ    │
                    │   Cluster     │
                    └───────┬───────┘
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
        ▼                   ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   MongoDB    │    │  PostgreSQL  │    │  SQL Server  │
│   Cluster    │    │   Cluster    │    │   Cluster    │
└──────────────┘    └──────────────┘    └──────────────┘
```

---

**This is a production-ready, event-driven microservices architecture with:**

- ✅ Complete user workflow implementation
- ✅ Saga pattern for distributed transactions
- ✅ Compensation/rollback support
- ✅ Event-driven choreography
- ✅ Multi-language microservices
- ✅ Comprehensive observability
- ✅ Security & authorization
- ✅ Scalability & resilience
