# Application Logging Guide

This guide describes how DigiOrg applications should emit logs to integrate well with the DigiOrg Core platform logging stack.

## Centralized Logging Overview

The DigiOrg Core platform collects, enriches, stores, and visualises all container logs automatically:

```
App stdout/stderr
      │
      ▼
/var/log/containers/*.log  (containerd log files on each node)
      │
      ▼
Fluentd DaemonSet  (digiorg/core#208)
  ├─ kubernetes_metadata_filter  →  enriches with namespace, pod, labels
  ├─ JSON parser  →  promotes structured log fields to top-level
  └─ out_elasticsearch  →  writes to OpenSearch
      │
      ▼
OpenSearch  (`digiorg-logs-*` index)  (digiorg/core#208, #216)
      │
      ▼
Grafana  (datasource: `OpenSearch (Logs)`, dashboard: DigiOrg Log Explorer)  (digiorg/core#212, #214)
```

Key facts:

- **No log agent required in your app.** Write to stdout/stderr; Fluentd handles everything else.
- **Kubernetes metadata is added automatically**: `kubernetes.namespace_name`, `kubernetes.pod_name`, `kubernetes.container_name`, pod labels, and annotations.
- **Index pattern**: `digiorg-logs-*` (daily rotation: `digiorg-logs-YYYY.MM.DD`).
- **Retention**: 7 days in the current local/dev platform setup (digiorg/core#216 OpenSearch ISM policy).

---

## Structured JSON Logging Contract

Fluentd collects all output regardless of format, but **structured JSON logs** are far easier to search, filter, and alert on in Grafana and OpenSearch. Plain-text logs are still collected but arrive as a single unsearchable `log` field.

### Recommended JSON shape

```json
{
  "timestamp": "2026-05-25T10:30:00.000Z",
  "level": "INFO",
  "message": "User login successful",
  "service": "my-app-api",
  "trace_id": "abc123def456",
  "span_id": "789xyz",
  "user_id": "u-42",
  "duration_ms": 45
}
```

### Required fields

| Field | Type | Description |
|---|---|---|
| `timestamp` | ISO-8601 string | Log timestamp with timezone (`Z` or `+HH:MM`) |
| `level` | string | Log level — one of `DEBUG`, `INFO`, `WARN`, `ERROR` (uppercase) |
| `message` | string | Human-readable description of the event |
| `service` | string | Service or app name; align with the AppClaim `spec.appName` |

### Recommended fields

| Field | Type | Description |
|---|---|---|
| `trace_id` | string | Jaeger trace ID — enables log ↔ trace correlation |
| `span_id` | string | Jaeger span ID |
| `duration_ms` | integer | Duration of the operation in milliseconds |
| `user_id` | string | User identifier (only if non-sensitive; see [Security](#security-and-privacy)) |
| `error` | string | Error message (include on `ERROR` level entries) |
| `stack_trace` | string | Stack trace (include on `ERROR` level entries when useful) |

Use stable, lowercase, underscore-separated field names. Avoid adding fields whose names change per request.

---

## Language Examples

### Node.js — pino

[pino](https://getpino.io) emits JSON by default and maps directly to the structured logging contract.

```javascript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  // pino uses 'msg' by default; rename to 'message' to match the contract
  messageKey: 'message',
  formatters: {
    level(label) { return { level: label.toUpperCase() }; },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

// Usage
logger.info({ service: 'my-app-api', traceId: span.traceId, userId: 'u-42', durationMs: 45 },
  'User login successful');

logger.error({ service: 'my-app-api', traceId: span.traceId, error: err.message, stackTrace: err.stack },
  'Database connection failed');
```

### Java — Logback + logstash-logback-encoder

Add the encoder dependency:

```xml
<!-- pom.xml -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>7.4</version>
</dependency>
```

Configure `logback.xml` to write JSON to stdout:

```xml
<configuration>
  <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <timestampPattern>yyyy-MM-dd'T'HH:mm:ss.SSS'Z'</timestampPattern>
      <fieldNames>
        <timestamp>timestamp</timestamp>
        <message>message</message>
        <logger>[ignore]</logger>
        <thread>[ignore]</thread>
        <version>[ignore]</version>
      </fieldNames>
      <customFields>{"service":"my-app-api"}</customFields>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="JSON_STDOUT"/>
  </root>
</configuration>
```

Add MDC values for trace correlation:

```java
import org.slf4j.MDC;

MDC.put("trace_id", traceId);
MDC.put("span_id", spanId);
log.info("User login successful");
MDC.clear();
```

### Python — structlog

```python
import structlog
import logging
import sys

structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    logger_factory=structlog.PrintLoggerFactory(file=sys.stdout),
)

logger = structlog.get_logger()

# Usage
logger.info("user_login_successful",
            service="my-app-api",
            trace_id=trace_id,
            user_id="u-42",
            duration_ms=45)

logger.error("database_connection_failed",
             service="my-app-api",
             trace_id=trace_id,
             error=str(exc))
```

### Go — zerolog

```go
import (
    "os"
    "time"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func init() {
    zerolog.TimeFieldFormat = time.RFC3339Nano
    zerolog.TimestampFieldName = "timestamp"
    zerolog.LevelFieldName = "level"
    zerolog.MessageFieldName = "message"
    zerolog.LevelFieldMarshalFunc = func(l zerolog.Level) string {
        return strings.ToUpper(l.String())
    }
    log.Logger = zerolog.New(os.Stdout).With().Timestamp().
        Str("service", "my-app-api").
        Logger()
}

// Usage
log.Info().
    Str("trace_id", traceID).
    Str("user_id", userID).
    Int("duration_ms", durationMs).
    Msg("User login successful")

log.Error().
    Str("trace_id", traceID).
    Err(err).
    Msg("Database connection failed")
```

---

## Operational Guidance

### Do

- **Log to stdout/stderr only.** Do not write to files; Fluentd reads from container stdout/stderr via the node's `/var/log/containers/` path.
- **Use uppercase log levels** (`DEBUG`, `INFO`, `WARN`, `ERROR`). Mixed case causes inconsistent filtering in OpenSearch.
- **Prefer stable field names.** Changing field names between deployments breaks saved Grafana queries and alerts.
- **Include `trace_id` and `span_id`** wherever a Jaeger span is in scope. This links log entries directly to distributed traces.
- **Set `service`** to the same value as `spec.appName` in your AppClaim so namespace-level and service-level filters align.

### Do not

- **Do not log secrets, tokens, passwords, session cookies, or private keys.** These appear in OpenSearch and in Fluentd buffer files on the node.
- **Do not log full PII payloads** (e.g. raw request bodies containing email addresses, national IDs, payment card numbers). Log identifiers (`user_id`) rather than raw data.
- **Do not use high-cardinality values as primary log fields** (e.g. a UUID per log line as a top-level key). Use a fixed field like `trace_id` instead.
- **Avoid excessive DEBUG logging in shared environments.** DEBUG output from all pods adds index size and makes ERROR searches noisier. Set `LOG_LEVEL=info` in production and staging.
- **Do not swallow errors silently.** An `ERROR` with `error` and `stack_trace` is far more useful than a missing log entry.

---

## Finding Logs in Grafana

### DigiOrg Log Explorer dashboard

1. Open Grafana → **Dashboards** → **DigiOrg Log Explorer**.
2. Use the top-bar filters to narrow by **namespace**, **pod**, **container**, and **level**.
3. The dashboard shows log volume over time, error rate per namespace, log-level breakdown, and a live log tail.

### Explore — ad-hoc queries

1. Grafana → **Explore** → select datasource **`OpenSearch (Logs)`**.
2. Enter a Lucene query in the query box.

Example queries:

| Goal | Query |
|---|---|
| All errors in your app's namespace | `kubernetes.namespace_name: my-app AND level: ERROR` |
| Trace a specific request end-to-end | `service: my-app-api AND trace_id: abc123def456` |
| All logs from a pod name prefix | `kubernetes.pod_name: my-app-*` |
| Warnings and errors only | `level: WARN OR level: ERROR` |
| Logs from a specific container | `kubernetes.container_name: my-app-api` |

### Tips

- The index pattern is `digiorg-logs-*`. If OpenSearch Explore shows no results, verify the time range covers a period when the app was running.
- Logs are retained for **7 days** in the local/dev platform. Queries outside that window return no results.
- For correlating a log entry with a trace, copy the `trace_id` value and paste it into Grafana → Explore → **Jaeger** datasource.
