# Standardized Logging Configuration Guide

## Overview

This document defines the standardized logging approach across all AIOutlet microservices to ensure consistent monitoring, debugging, and observability.

## Log Levels and Usage

### Development Environment

- **DEBUG**: Detailed flow information, variable states, method entry/exit
- **INFO**: General application flow, major operations, service startup/shutdown
- **WARN**: Potentially harmful situations, configuration issues, deprecated usage
- **ERROR**: Error events that don't stop the application
- **FATAL/CRITICAL**: Serious failures that will abort the application

### Production Environment

- **INFO**: Business operations, request completions, major state changes
- **WARN**: Performance issues, configuration warnings, fallback usage
- **ERROR**: Error events that need attention but don't stop service
- **FATAL/CRITICAL**: Serious failures requiring immediate attention

## Standard Log Format

### Required Fields

Every log entry must include:

- `timestamp`: ISO 8601 format
- `level`: Log level (DEBUG, INFO, WARN, ERROR, FATAL)
- `service`: Service name (e.g., "user-service", "order-service")
- `correlationId`: Request correlation ID for distributed tracing
- `message`: Human-readable description

### Optional Fields

- `userId`: User identifier when available
- `operation`: Business operation being performed
- `duration`: Operation duration in milliseconds
- `error`: Error details and stack trace
- `metadata`: Additional context-specific data

### Development Format (Console)

```
[2025-08-12T10:30:45.123Z] [INFO] user-service [abc-123]: User created successfully | userId=12345, operation=CREATE_USER, duration=45ms
```

### Production Format (JSON)

```json
{
  "timestamp": "2025-08-12T10:30:45.123Z",
  "level": "INFO",
  "service": "user-service",
  "correlationId": "abc-123",
  "message": "User created successfully",
  "userId": "12345",
  "operation": "CREATE_USER",
  "duration": 45,
  "metadata": {}
}
```

## Environment Configuration

### Environment Variables

- `NODE_ENV` / `ASPNETCORE_ENVIRONMENT` / `GO_ENV`: development, staging, production
- `LOG_LEVEL`: debug, info, warn, error
- `LOG_FORMAT`: console, json
- `LOG_TO_FILE`: true/false
- `LOG_FILE_PATH`: Path to log file
- `LOG_TO_CONSOLE`: true/false

### Per-Service Defaults

- **Development**: Console output with colors, DEBUG level
- **Production**: JSON format to file/stdout, INFO level
- **Testing**: Silent or minimal logging

## Service-Specific Implementation

### Node.js Services (Auth, User, Admin)

- Winston logger with custom formatters
- Correlation ID integration
- Environment-based configuration
- Color output for development

### Python Services (Product)

- Python logging with custom formatters
- Structured logging with extra fields
- JSON output for production

### Go Services (Inventory)

- Logrus with structured logging
- Correlation ID middleware integration
- JSON formatter for production

### C# Services (Order, Payment)

- Serilog with structured logging
- ASP.NET Core logging integration
- Correlation ID enrichment

## Security Considerations

### Sensitive Data Protection

Never log:

- Passwords or password hashes
- Credit card numbers
- Social security numbers
- API keys or tokens (log only first/last few characters)
- Personal data without user consent

### Log Sanitization

- Mask or redact sensitive fields
- Use placeholders for sensitive operations
- Implement log scrubbing for compliance

## Performance Guidelines

### High-Performance Logging

- Use async logging in production
- Implement log rotation to prevent disk space issues
- Use appropriate log levels to reduce volume
- Consider sampling for high-volume operations

### Log Retention

- Development: 7 days
- Staging: 30 days
- Production: 90 days (or per compliance requirements)

## Monitoring Integration

### Log Aggregation

- Centralized logging with ELK Stack, Splunk, or similar
- Correlation ID indexing for request tracing
- Alerting on ERROR and FATAL logs

### Metrics Integration

- Log-based metrics for business operations
- Performance metrics from duration fields
- Error rate tracking from log levels
