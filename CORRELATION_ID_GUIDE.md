# Correlation ID Implementation Guide

## Overview

This document describes the end-to-end correlation ID implementation across all microservices in the AIOutlet platform. Correlation IDs enable distributed tracing, allowing you to track a single request as it flows through multiple services.

## What is a Correlation ID?

A correlation ID is a unique identifier (UUID) that is:

- Generated for each incoming request (or passed through if already exists)
- Propagated through all service-to-service communications
- Included in all log messages
- Returned in response headers

## Implementation Status

### ✅ Implemented Services

| Service           | Language/Framework | Port | Status      |
| ----------------- | ------------------ | ---- | ----------- |
| Order Service     | C#/.NET Core       | 7000 | ✅ Complete |
| Payment Service   | C#/.NET Core       | 8080 | ✅ Complete |
| Auth Service      | Node.js/Express    | 4000 | ✅ Complete |
| User Service      | Node.js/Express    | 5000 | ✅ Complete |
| Admin Service     | Node.js/Express    | 6000 | ✅ Complete |
| Product Service   | Python/FastAPI     | 8000 | ✅ Complete |
| Inventory Service | Go/Gin             | 3000 | ✅ Complete |

## Architecture

### Middleware Implementation

Each service implements correlation ID middleware that:

1. Extracts correlation ID from `X-Correlation-ID` header
2. Generates new UUID if header is missing
3. Stores correlation ID in request context
4. Adds correlation ID to response headers
5. Logs request with correlation ID

### Header Standard

- **Header Name**: `X-Correlation-ID`
- **Format**: UUID v4 (e.g., `550e8400-e29b-41d4-a716-446655440000`)
- **Direction**: Bidirectional (request and response)

## Service-Specific Implementation

### Node.js Services (Auth, User, Admin)

```javascript
// Middleware
import correlationIdMiddleware from './middlewares/correlationId.middleware.js';
app.use(correlationIdMiddleware);

// Usage in controllers
import CorrelationIdHelper from '../utils/correlationId.helper.js';
const correlationId = CorrelationIdHelper.getCorrelationId(req);

// Inter-service calls
import { userServiceClient } from '../utils/serviceClient.js';
const response = await userServiceClient.get(req, '/api/users/123');
```

### Python Service (Product)

```python
# Middleware
from src.middlewares.correlation_id import CorrelationIdMiddleware
app.add_middleware(CorrelationIdMiddleware)

# Usage in endpoints
from src.middlewares.correlation_id import get_correlation_id
correlation_id = get_correlation_id()

# Inter-service calls
from src.utils.service_client import user_service_client
response = await user_service_client.get('/api/users/123')
```

### Go Service (Inventory)

```go
// Middleware
router.Use(middleware.CorrelationIDMiddleware())

// Usage in handlers
correlationID := middleware.GetCorrelationID(c)

// Inter-service calls
client := client.NewServiceClient("http://user-service:5000", 5*time.Second, logger)
response, err := client.GetWithGinContext(c, "/api/users/123")
```

### C# Services (Order, Payment)

```csharp
// Middleware
app.UseCorrelationId();

// Usage in controllers
var correlationId = HttpContext.Items["CorrelationId"]?.ToString();

// Inter-service calls (Payment Service has CorrelationIdHelper)
var correlationId = CorrelationIdHelper.GetCorrelationId();
```

## Testing Correlation ID

### Manual Testing

```bash
# Generate a test correlation ID
CORRELATION_ID=$(uuidgen)

# Test with curl
curl -H "X-Correlation-ID: $CORRELATION_ID" \
     -H "Content-Type: application/json" \
     http://localhost:7000/api/orders

# Check logs for the correlation ID
grep "$CORRELATION_ID" *.log
```

### Automated Testing

Run the included test script:

```bash
chmod +x scripts/test-correlation-id.sh
./scripts/test-correlation-id.sh
```

## End-to-End Tracing Example

### Typical Request Flow

1. **Client** → Order Service (generates correlation ID: `abc-123`)
2. **Order Service** → Inventory Service (forwards: `abc-123`)
3. **Inventory Service** → Product Service (forwards: `abc-123`)
4. **Product Service** → User Service (forwards: `abc-123`)

### Log Output Example

```
[abc-123] Order Service: Processing order creation request
[abc-123] Inventory Service: Checking stock for product 456
[abc-123] Product Service: Fetching product details for 456
[abc-123] User Service: Validating user permissions
[abc-123] Product Service: Product details retrieved successfully
[abc-123] Inventory Service: Stock available, reserving items
[abc-123] Order Service: Order created successfully
```

## Best Practices

### 1. Always Propagate Correlation IDs

- Use provided service clients for inter-service communication
- Never make direct HTTP calls without correlation ID headers
- Ensure correlation ID is passed to async operations

### 2. Consistent Logging

```javascript
// Good - includes correlation ID
logger.info('User created', {
  userId: user.id,
  correlationId: CorrelationIdHelper.getCorrelationId(req),
});

// Bad - missing correlation ID
logger.info('User created', { userId: user.id });
```

### 3. Error Handling

Include correlation ID in error responses:

```javascript
res.status(500).json({
  error: 'Internal server error',
  correlationId: CorrelationIdHelper.getCorrelationId(req),
  timestamp: new Date().toISOString(),
});
```

### 4. Database Operations

Consider adding correlation ID to database audit logs:

```javascript
const auditLog = {
  action: 'CREATE_USER',
  userId: user.id,
  correlationId: CorrelationIdHelper.getCorrelationId(req),
  timestamp: new Date(),
};
```

## Monitoring and Observability

### Log Aggregation

Configure your log aggregation system (ELK, Splunk, etc.) to:

- Index correlation ID field
- Create dashboards for request flow visualization
- Set up alerts for failed transactions by correlation ID

### Performance Monitoring

Track request latency across services using correlation IDs:

```bash
# Find slow requests
grep "abc-123" *.log | grep "duration"
```

### Debugging

When investigating issues:

1. Get correlation ID from error report
2. Search all service logs for that correlation ID
3. Reconstruct the complete request flow
4. Identify the point of failure

## Troubleshooting

### Common Issues

#### Missing Correlation ID in Logs

- Check middleware order (correlation ID should be first)
- Verify logger configuration includes correlation ID
- Ensure helper methods are used correctly

#### Correlation ID Not Propagated

- Verify service client implementation
- Check HTTP header names (case-sensitive)
- Ensure context is passed correctly in async operations

#### Performance Impact

- Correlation ID processing is lightweight
- UUID generation is fast
- Header processing has minimal overhead

### Debugging Commands

```bash
# Test all services
./scripts/test-correlation-id.sh

# Check specific service
curl -H "X-Correlation-ID: test-123" http://localhost:5000/api/home

# Find correlation ID in logs
grep -r "test-123" logs/

# Verify middleware is loaded
# Check startup logs for correlation ID middleware registration
```

## Future Enhancements

### Planned Improvements

1. **Distributed Tracing**: Integrate with Jaeger or Zipkin
2. **Metrics**: Add correlation ID to Prometheus metrics
3. **APM Integration**: Connect with New Relic/DataDog
4. **Request Batching**: Handle bulk operations with sub-correlation IDs

### OpenTelemetry Integration

Consider migrating to OpenTelemetry for:

- Standardized tracing
- Automatic instrumentation
- Better observability tooling
- Industry-standard trace formats

## Summary

The correlation ID implementation provides:

- ✅ Complete request traceability across all 7 microservices
- ✅ Consistent logging with correlation context
- ✅ Service-to-service communication helpers
- ✅ Automated testing capabilities
- ✅ Developer-friendly debugging tools

This foundation enables effective monitoring, debugging, and performance analysis in your distributed microservices architecture.
