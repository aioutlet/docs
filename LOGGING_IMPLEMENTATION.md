# Standardized Logging Implementation Guide

## Overview

This document outlines the standardized logging implementation across all microservices in the AI Outlet platform. The logging system is designed to provide consistent, correlation ID-enabled, environment-aware logging with proper dev vs production configurations.

## Services Covered

1. **Node.js Services** (Express/JavaScript):

   - auth-service
   - user-service
   - admin-service

2. **Python Service** (FastAPI):

   - product-service

3. **Go Service** (Gin):

   - inventory-service

4. **C# Services** (.NET Core):
   - order-service
   - payment-service

## Key Features

### 1. Correlation ID Integration

- All log entries include correlation ID for distributed tracing
- Correlation ID flows through all service calls
- Enables end-to-end request tracing across microservices

### 2. Environment-Based Configuration

- **Development**: Console output with colored formatting, debug level logging
- **Production**: JSON structured logging, info level, file output

### 3. Standardized Log Levels

- **Debug**: Detailed debugging information (dev only)
- **Info**: General informational messages
- **Warn**: Warning conditions
- **Error**: Error conditions
- **Fatal**: Critical errors

### 4. Operation Tracking

- Operation start/complete/failed logging
- Automatic duration tracking
- Performance metrics

### 5. Business Event Logging

- Business event tracking
- Security event logging
- Performance monitoring

## Implementation Details

### Node.js Services (Winston-based)

**Location**: `src/utils/standardLogger.js`

```javascript
const standardLogger = require('./utils/standardLogger');

// Usage examples
standardLogger.info('User login attempt', { userId: '123' });
const startTime = standardLogger.operationStart('user_creation');
// ... operation logic
standardLogger.operationComplete('user_creation', startTime, { userId: newUser.id });
```

**Configuration Files**:

- `.env.development`: Console logging, debug level
- `.env.production`: JSON logging, info level

### Python Service (Custom Logger)

**Location**: `src/core/standard_logger.py`

```python
from src.core.standard_logger import StandardLogger

logger = StandardLogger()

# Usage examples
logger.info("Product search initiated", {"query": "laptop"})
start_time = logger.operation_start("product_search")
# ... operation logic
logger.operation_complete("product_search", start_time, {"results": len(products)})
```

### Go Service (Logrus-based)

**Location**: `internal/logger/standard_logger.go`

```go
import "inventory-service/internal/logger"

standardLogger := logger.NewStandardLogger()

// Usage examples
standardLogger.Info("Stock check initiated", c, map[string]interface{}{
    "sku": "LAPTOP001",
})
startTime := standardLogger.OperationStart("stock_check", c, nil)
// ... operation logic
standardLogger.OperationComplete("stock_check", startTime, c, metadata)
```

### C# Services (Serilog-based)

**Location**: `Utils/StandardLogger.cs`

```csharp
using OrderService.Utils;

private readonly StandardLogger _logger;

// Usage examples
_logger.Info("Order processing started", correlationId, new { orderId = order.Id });
var stopwatch = _logger.OperationStart("process_order", correlationId);
// ... operation logic
_logger.OperationComplete("process_order", stopwatch, correlationId, metadata);
```

## Environment Configuration

### Development Environment

- **Log Level**: Debug
- **Format**: Console with colors and structured data
- **Output**: Console only
- **Features**: Detailed error messages, stack traces

### Production Environment

- **Log Level**: Info
- **Format**: JSON structured logging
- **Output**: Console + File (where applicable)
- **Features**: Structured data for log aggregation, minimal verbosity

## Log Structure

### Standard Log Entry Format

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "service": "user-service",
  "correlationId": "abc123-def456-ghi789",
  "message": "User authentication successful",
  "operation": "user_login",
  "duration": 150,
  "userId": "user_123",
  "metadata": {
    "userAgent": "Mozilla/5.0...",
    "ip": "192.168.1.100"
  }
}
```

### Key Fields

- **timestamp**: ISO 8601 timestamp
- **level**: Log level (debug, info, warn, error, fatal)
- **service**: Service name identifier
- **correlationId**: Request correlation ID
- **message**: Human-readable message
- **operation**: Operation being performed
- **duration**: Operation duration in milliseconds
- **metadata**: Additional contextual data

## Log Categories

### 1. HTTP Request Logs

- Method, path, status code, duration
- Client IP, User Agent
- Request/response correlation

### 2. Operation Logs

- Operation start/complete/failed
- Duration tracking
- Business context

### 3. Business Event Logs

- User registrations, logins
- Order creation, payment processing
- Product updates, inventory changes

### 4. Security Event Logs

- Authentication failures
- Authorization violations
- Suspicious activity

### 5. Performance Logs

- Slow queries
- High memory usage
- API response times

## Usage Guidelines

### 1. Correlation ID Usage

- Always pass correlation ID to logger methods
- Extract from HTTP headers (`X-Correlation-ID`)
- Propagate through service calls

### 2. Metadata Best Practices

- Include relevant business context
- Avoid sensitive information (passwords, tokens)
- Use consistent field names across services

### 3. Log Level Guidelines

- **Debug**: Development debugging only
- **Info**: Normal operation events
- **Warn**: Unusual but not error conditions
- **Error**: Error conditions that don't stop execution
- **Fatal**: Critical errors that require immediate attention

### 4. Performance Considerations

- Use appropriate log levels in production
- Avoid logging large payloads
- Consider async logging for high-throughput scenarios

## Testing the Implementation

### 1. Correlation ID Flow Test

```bash
# Test correlation ID propagation
curl -H "X-Correlation-ID: test-123" http://localhost:3001/api/users/profile
# Check logs across all services for correlation ID "test-123"
```

### 2. Environment Configuration Test

```bash
# Test development logging
ENVIRONMENT=development npm start

# Test production logging
ENVIRONMENT=production npm start
```

### 3. Log Structure Validation

- Verify JSON format in production
- Check all required fields are present
- Validate correlation ID propagation

## Monitoring and Alerting

### 1. Log Aggregation

- Recommended: ELK Stack (Elasticsearch, Logstash, Kibana)
- Alternative: Grafana Loki, Fluentd
- Cloud: Azure Monitor, AWS CloudWatch

### 2. Key Metrics to Monitor

- Error rate by service
- Average response times
- Failed operations count
- Security event frequency

### 3. Alerting Rules

- Error rate > threshold
- Critical/Fatal log events
- Missing correlation IDs
- Service availability issues

## Troubleshooting

### Common Issues

1. **Missing Correlation IDs**

   - Check middleware configuration
   - Verify header propagation

2. **Inconsistent Log Formats**

   - Verify environment configuration
   - Check logger initialization

3. **Performance Issues**
   - Review log level settings
   - Consider async logging
   - Monitor disk space for file logging

### Debug Commands

```bash
# Check environment configuration
echo $ENVIRONMENT

# Test logger initialization
npm test -- --grep "logger"

# Validate log format
tail -f /var/log/service.log | jq .
```

## Future Enhancements

1. **Distributed Tracing Integration**

   - OpenTelemetry support
   - Jaeger integration
   - Span correlation

2. **Advanced Alerting**

   - ML-based anomaly detection
   - Predictive alerting
   - Auto-scaling triggers

3. **Log Analytics**
   - Business intelligence dashboards
   - Performance trend analysis
   - User behavior tracking

## Conclusion

The standardized logging implementation provides:

- Consistent logging across all microservices
- Correlation ID-based distributed tracing
- Environment-appropriate configurations
- Structured logging for analysis
- Performance and business event tracking

This foundation enables effective monitoring, debugging, and business intelligence across the entire AI Outlet platform.
