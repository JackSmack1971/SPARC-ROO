# DevOps Engineer Rule: 03 - System Observability Framework

**Purpose:** To establish a unified strategy for system monitoring, logging, and tracing. This framework moves beyond reactive monitoring to proactive **Observability**, enabling us to debug complex issues in a distributed environment, manage system health against user-centric objectives, and make data-driven decisions.

---

### 1. Philosophy: From Monitoring to Observability

* **Monitoring** is watching for *known* failures (e.g., "Is the CPU over 90%?").
* **Observability** is building systems that can answer questions we don't know we need to ask yet (e.g., "Why are API requests for users in Germany on Android devices 500ms slower?").

Our approach is centered on **Service Level Objectives (SLOs)**. We measure what matters to the user—**availability, latency, and correctness**—not just internal system metrics. An alert signifies a real or imminent impact on the user experience.

---

### 2. The Three Pillars of Observability

Our framework is built on three distinct but interconnected types of telemetry data. Every service **MUST** emit data for all three pillars.



#### Pillar 1: Structured Logging
Logs provide detailed, event-specific context. They are the "why" behind an issue.

* **Standard:** All logs **MUST** be emitted as structured **JSON** to `stdout`. This makes them machine-parsable, searchable, and easy to aggregate.
* **Content:** Every log entry **MUST** contain a `traceId` to correlate it with a specific request's journey through our system.
* **Implementation (`logger.ts`):** We use a standardized logger utility built on a performant library like `pino`.

```typescript
// src/lib/logger.ts
import pino from 'pino';
import { context, trace } from '@opentelemetry/api';

// Create a base logger instance
const baseLogger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
});

/**
 * A logger that automatically injects the active traceId and spanId
 * into every log message for correlation.
 */
export const logger = {
  info: (msg: string, payload?: Record<string, any>) => {
    const span = trace.getSpan(context.active());
    if (span) {
      const { traceId, spanId } = span.spanContext();
      baseLogger.info({ traceId, spanId, ...payload }, msg);
    } else {
      baseLogger.info(payload, msg);
    }
  },
  error: (msg: string, error?: Error, payload?: Record<string, any>) => {
    const span = trace.getSpan(context.active());
    if (span) {
      const { traceId, spanId } = span.spanContext();
      baseLogger.error({ traceId, spanId, err: error, ...payload }, msg);
    } else {
      baseLogger.error({ err: error, ...payload }, msg);
    }
  },
  // ... other levels (warn, debug)
};
````

#### Pillar 2: Metrics (The RED Method)

Metrics are aggregated numerical data that provide a high-level view of system health and performance over time.

  * **Standard:** For every service and API endpoint, we **MUST** track the **RED** metrics:
      * **R**ate: The number of requests per second.
      * **E**rrors: The number of failing requests per second.
      * **D**uration: The distribution of time each request takes (we track p50, p90, p95, p99 percentiles).
  * **Implementation:** We use **Prometheus** for metrics collection and a library like `prom-client` to expose a `/metrics` endpoint on each service.

<!-- end list -->

```typescript
// src/middleware/metricsMiddleware.ts
import { Request, Response, NextFunction } from 'express';
import { Counter, Histogram } from 'prom-client';

const requestCounter = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

const requestDurationHistogram = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Histogram of request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.5, 1, 1.5, 5], // Buckets in seconds
});

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();
  res.on('finish', () => {
    const durationSeconds = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : 'unknown';
    
    requestCounter.inc({
      method: req.method,
      route: route,
      status_code: res.statusCode,
    });
    
    requestDurationHistogram.observe({
      method: req.method,
      route: route,
    }, durationSeconds);
  });
  next();
}
```

#### Pillar 3: Distributed Tracing

Traces provide a detailed view of a single request's lifecycle as it travels through multiple services, essential for debugging latency and errors in a microservices architecture.

  * **Standard:** All services **MUST** participate in distributed tracing using the **OpenTelemetry** standard.
  * **Implementation:** The OpenTelemetry SDK is configured once in each service's entry point to automatically instrument common libraries (Express, Prisma, `fetch`, etc.). The API Gateway initiates the trace and propagates the context via HTTP headers.

<!-- end list -->

```typescript
// src/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';

// Configure the SDK to send traces to an OTLP-compatible collector (e.g., Jaeger, Grafana Tempo)
const otlpExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
});

const sdk = new NodeSDK({
  traceExporter: otlpExporter,
  instrumentations: [getNodeAutoInstrumentations()],
  // Add service name and other resources here
});

sdk.start();

// Ensure graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => console.log('Tracing terminated.'));
});
```

-----

### 3\. SLOs, Alerting, and Dashboards

  * **Service Level Objectives (SLOs):** We define explicit, user-centric reliability targets for each critical service. The **SPARC-Architect** is responsible for defining these SLOs.
      * **Example SLO:** "The User Service `GET /api/v1/users/{id}` endpoint will have 99.9% availability over a 28-day rolling window."
  * **Alerting Philosophy:** We alert on **SLO violations**, not arbitrary system thresholds. An alert must be actionable and indicate a real problem.
      * **Good Alert:** "API latency p99 has exceeded 500ms for 5 minutes, burning our error budget at 20x the normal rate."
      * **Bad Alert (to be avoided):** "CPU usage is at 80%."
  * **Dashboards:** We use **Grafana** to build standardized dashboards for each service, providing a unified view of the three pillars and SLO status.

-----

### 4\. Memory Bank Integration: Operational Runbooks

Every actionable alert **MUST** have a corresponding runbook documented in the Memory Bank. This turns an alert from a moment of panic into a structured, debuggable event.

**Memory Bank Template: Runbook (`runbook.yml`)**

```yaml
#---
# File: memory/devops-engineer/runbooks/high-api-error-rate-payment-service.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: devops-engineer
  layer: operations
  title: "Runbook: High API Error Rate on Payment Service"
  author: "SPARC-DevOps-Engineer"
  timestamp: "2025-08-06T00:25:00Z"
spec:
  summary: "This runbook provides triage and resolution steps for the PagerDuty alert 'High 5xx Error Rate on Payment Service'."
  type: "Runbook"
  
  content: |
    ### Alert Definition:
    - **Source:** Prometheus Alertmanager
    - **Condition:** `(sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m])) / sum(rate(http_requests_total{service="payment-service"}[5m]))) > 0.05`
    - **Meaning:** More than 5% of requests to the Payment Service have failed in the last 5 minutes.

    ### Triage Steps (First 5 Minutes):
    1.  **Check the Dashboard:** Open the "Payment Service" Grafana dashboard.
    2.  **Identify Failing Endpoint:** Look at the "Requests by Endpoint" panel to see which specific API call is failing.
    3.  **Examine Logs:** In Grafana Loki/Explore, filter logs by `service="payment-service"` and `level="error"`. Look for recent error messages.
    4.  **Inspect Traces:** Find a few failed `traceId`s from the logs and view them in Jaeger/Tempo to see which downstream dependency is failing (e.g., database, external Stripe API).

    ### Common Causes & Resolutions:
    - **Cause:** External Stripe API is down.
      - **Resolution:** Check the Stripe status page. If confirmed, post an incident update and monitor. The system should degrade gracefully.
    - **Cause:** Invalid database credentials after a rotation.
      - **Resolution:** Trigger a redeploy of the service from the main branch to pull the latest secrets from Vault/Secrets Manager.
    - **Cause:** A recent deployment introduced a bug.
      - **Resolution:** If the issue correlates with the last deployment time, initiate a rollback via the pipeline's "Blue-Green" rollback action.

  associations:
    - "slo-payment-service-availability"
  
  status: "Active"
```
