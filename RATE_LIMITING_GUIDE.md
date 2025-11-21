# Dapr Rate Limiting Configuration Example

To implement rate limiting using Dapr instead of custom .NET code, add this to `.dapr/components/rate-limit.yaml`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: rate-limit
spec:
  type: middleware.http.ratelimit
  version: v1
  metadata:
    # Maximum requests allowed
    - name: maxRequestsPerSecond
      value: "100"
```

Then reference it in `.dapr/config.yaml`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: payment-service-config
spec:
  httpPipeline:
    handlers:
      - name: rate-limit
        type: middleware.http.ratelimit
  features:
    - name: SchedulerReminders
      enabled: false
  tracing:
    samplingRate: '1'
    otel:
      endpointAddress: 'localhost:4318'
      isSecure: false
      protocol: 'http'
```

## Benefits of Dapr Rate Limiting:
- Configured outside application code
- Consistent across all language SDKs
- No code changes needed to adjust limits
- Applied at sidecar level (more efficient)
- GitOps-friendly
