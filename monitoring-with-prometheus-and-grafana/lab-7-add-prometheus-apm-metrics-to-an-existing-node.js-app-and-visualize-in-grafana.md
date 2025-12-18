# Lab 7: Add Prometheus APM Metrics to an Existing Node.js App and Visualize in Grafana

### Overview

In this lab, you will instrument an existing Node.js application with Prometheus-compliant metrics using `prom-client`, expose a `/metrics` endpoint, configure Prometheus to scrape the app, and build APM panels in Grafana (Rate, Errors, Duration, plus basic process metrics).

### Learning Objectives

By the end, you can:

* Add Prometheus instrumentation to an existing Node.js app using `prom-client`
* Expose `/metrics` safely and validate output
* Configure Prometheus scraping for the Node app
* Build Grafana APM panels using PromQL (RPS, Error %, P95 latency, Top routes)

### Prerequisites

* An existing Node.js app running locally or on a VM/container
* Node.js and npm installed
* Prometheus running (Docker or binary)
* Grafana running and connected to Prometheus

### What you will build

* Node.js app exposes: `GET /metrics`
* Prometheus scrapes: `http://<app-host>:<app-port>/metrics`
* Grafana dashboards visualize:
  * Request rate (RPS)
  * Error rate (%)
  * Latency (P95)
  * Top routes
  * Node process metrics (memory, CPU)

### Task 1: Add Prometheus client library

From your application root:

```bash
npm i prom-client
```

If TypeScript:

```bash
npm i -D @types/prom-client
```

### Task 2: Create a metrics module

Create a file: `src/metrics.js`

```js
const client = require("prom-client");

// Collect default Node.js metrics (CPU, memory, GC, event loop, etc)
client.collectDefaultMetrics();

// Request counter
const httpRequestsTotal = new client.Counter({
  name: "http_requests_total",
  help: "Total number of HTTP requests",
  labelNames: ["method", "route", "status_code"]
});

// Duration histogram (APM latency)
const httpRequestDurationSeconds = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "Duration of HTTP requests in seconds",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2, 5]
});

// In-flight requests gauge
const httpRequestsInFlight = new client.Gauge({
  name: "http_requests_in_flight",
  help: "Number of HTTP requests currently being processed"
});

function metricsMiddleware(req, res, next) {
  const endTimer = httpRequestDurationSeconds.startTimer();
  httpRequestsInFlight.inc();

  res.on("finish", () => {
    const route = (req.route && req.route.path) ? req.route.path : req.path;

    const labels = {
      method: req.method,
      route,
      status_code: String(res.statusCode)
    };

    httpRequestsTotal.inc(labels);
    endTimer(labels);
    httpRequestsInFlight.dec();
  });

  next();
}

async function metricsEndpoint(req, res) {
  res.setHeader("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
}

module.exports = {
  metricsMiddleware,
  metricsEndpoint
};
```

Notes:

* `req.route.path` helps reduce high-cardinality metrics for routes like `/users/:id`.
* If your app is not Express, keep this file and adjust the middleware wiring step in Task 3.

### Task 3: Wire metrics into your existing Node app (Express example)

Open your main server file, usually one of these:

* `app.js`
* `server.js`
* `index.js`
* `src/app.js`

Add the metrics middleware and endpoint.

```js
const express = require("express");
const { metricsMiddleware, metricsEndpoint } = require("./src/metrics");

const app = express();

// Add metrics middleware early so it measures all routes
app.use(metricsMiddleware);

// Your existing routes
app.get("/health", (req, res) => res.send("ok"));

// Metrics endpoint
app.get("/metrics", metricsEndpoint);

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`App listening on ${port}`));
```

Security guidance for real environments:

* Keep `/metrics` internal-only (private network, firewall rules, or a sidecar)
* Or protect `/metrics` using basic auth or an IP allowlist

### Task 4: Run and validate metrics output

Start your app:

```bash
npm start
```

Hit a few endpoints:

```bash
curl -s http://localhost:3000/health
curl -s http://localhost:3000/health
```

Check metrics:

```bash
curl -s http://localhost:3000/metrics | head -n 30
curl -s http://localhost:3000/metrics | grep http_requests_total | head
```

Expected:

* You can see `http_requests_total`
* You can see histogram series like `http_request_duration_seconds_bucket`
* You can see default Node metrics like `process_resident_memory_bytes`

### Task 5: Configure Prometheus to scrape the Node app

Edit `prometheus.yml` and add a scrape job:

```yaml
scrape_configs:
  - job_name: "node-app"
    metrics_path: /metrics
    static_configs:
      - targets: ["localhost:3000"]
```

If Prometheus runs in Docker and your app runs on the host:

* On Mac/Windows use `host.docker.internal:3000`
* On Linux you may need host networking or the host gateway approach

Restart Prometheus.

Docker example:

```bash
docker restart prometheus
```

### Task 6: Validate metrics in Prometheus

Open Prometheus UI, then query:

Request count:

```promql
http_requests_total
```

Request rate:

```promql
sum(rate(http_requests_total[1m]))
```

P95 latency:

```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

If you see time series updating when you hit the app endpoints, scraping is working.

### Task 7: Build Grafana APM dashboard

Create a dashboard: **Node.js APM (Prometheus)**

#### Panel 1: Request rate (RPS)

```promql
sum(rate(http_requests_total[1m]))
```

#### Panel 2: Error rate percentage (5xx)

```promql
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m]))
```

#### Panel 3: P95 latency

```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

#### Panel 4: Top routes by traffic

```promql
topk(10, sum(rate(http_requests_total[5m])) by (route))
```

#### Panel 5: In-flight requests

```promql
sum(http_requests_in_flight)
```

#### Panel 6: Node process memory

```promql
process_resident_memory_bytes
```

Optional panels:

* CPU (process):

```promql
rate(process_cpu_seconds_total[1m])
```

***

### Task 8: Generate load and observe behavior

Run these to create meaningful charts:

```bash
for i in {1..200}; do curl -s http://localhost:3000/health >/dev/null; done
```

If you have an endpoint that triggers errors, call it repeatedly and watch the error panel.

### Troubleshooting

* Prometheus target is DOWN:
  * Confirm app is reachable from Prometheus network namespace
  * Confirm `/metrics` returns 200 and plaintext output
* Metrics exist but route labels look wrong:
  * Ensure middleware is registered before routes
  * For routers, ensure `req.route.path` is available for your framework
* Too many unique `route` values:
  * Avoid using raw `req.path` if it includes IDs; prefer templated routes
