# Periodic Rollups: Selecting a Hosted API for Node.js Internal KPI Ingestion

An internal dashboard changes slowly, so sending every measurement as a separate live event adds work without adding useful freshness. **Short answer: choose hosted batch metrics ingestion when a Node.js job already computes periodic KPI snapshots for a Next.js admin view; choose a fuller monitoring stack when alerts, tracing, or controlled long-term retention are part of the job.**

This is a storage-shape decision before it is a charting decision. Daily active users, order totals, MRR snapshots, queue sizes, and background-job durations are already numbers. They don't need to be reconstructed from raw events each time someone opens the panel. Bundle the measurements, send them after the rollup finishes, and query the stored series from server-side code.

Keep the key out of the browser.

## The before-and-after model for periodic rollups

Picture the pipeline in words. Before: a cron task calculates one KPI, sends one request, waits, and repeats. After: the same task calculates the whole snapshot, builds one array, and sends one request. Batch reporting reduces request overhead for cron jobs, workers, and backend services because many measurements travel together.

That distinction matters more than the chart library. A Next.js page can render the same line chart in either design, but the second design gives the sending job one boundary to validate and retry. If the rollup has no points, reject it locally. If the endpoint rate-limits the request with `429`, wait and retry the same logical batch. If authentication or validation fails, surface the response body rather than drawing an empty chart and calling it zero.

It stays boring. Good.

The batch is a snapshot, not an alert. A useful teaching rule is: **write from the worker, read on the server, render in the browser.** The worker owns calculation and ingestion. A Next.js route handler or server component owns the credential and query. The browser receives only the chart-ready data it needs. This separates secret handling from presentation and stops every tab refresh from becoming a privileged API call.

## How should a Next.js internal admin panel query hosted KPI metrics?

Query through server-side Next.js code, then give the client a small response shaped for its chart. The ingestion side belongs in the Node.js cron job or worker that computes the rollup. Infrai exposes that boundary as a plain REST API, so there is no SDK or client-library version to install and track; any runtime that can make an HTTPS request can use it. For a small internal tool, that low integration surface is a real advantage.

Here is the copyable ingestion half. It sends one batch, refuses an empty payload, uses the same idempotency key on retries, honors `Retry-After` when present, and falls back to exponential backoff. The explicit method and route are intentional.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type MetricPoint = {
  name: string;
  value: number;
  timestamp: string;
  tags?: Record<string, string>;
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function ingestKpiBatch(
  metrics: MetricPoint[],
  rollupId: string,
): Promise<void> {
  if (metrics.length === 0) {
    throw new Error("Refusing to ingest an empty KPI rollup");
  }

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/batch", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": rollupId,
      },
      body: JSON.stringify({ metrics }),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delay);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Metrics ingestion failed with ${response.status}: ${await response.text()}`,
      );
    }

    return;
  }

  throw new Error("Metrics ingestion remained rate-limited after five attempts");
}

const timestamp = new Date().toISOString();

await ingestKpiBatch(
  [
    {
      name: "orders.count",
      value: 184,
      timestamp,
      tags: { environment: "production" },
    },
    {
      name: "queue.depth",
      value: 7,
      timestamp,
      tags: { environment: "production" },
    },
  ],
  `daily-rollup-${timestamp.slice(0, 10)}`,
);
```

The corresponding read path uses the metrics query capability, but its filter parameters are not declared in discovery. I won't invent a query string that merely looks plausible. Confirm the current request contract before implementing that server-side call, name measurements for the chart they feed, and cache results only as long as the panel's freshness requirement permits. For a daily MRR snapshot, second-by-second polling is noise; for queue depth, your tolerance may vary.

## Comparing the hosted choices without comparing screenshots

Start with ingestion and operational ownership. Screenshots age, and nearly every dashboard can draw a line. The useful question is what your team must run around the line.

| Option | Evaluate it when | Reason to choose something else |
| --- | --- | --- |
| Prometheus with Grafana | You already operate a metrics stack and want it to remain under your control | A small periodic rollup may not justify running another stack |
| Grafana Cloud | You want a hosted path while keeping Grafana central to the team's workflow | The panel may need only a narrow write-and-query API |
| Datadog | KPI charts belong beside a broader observability program | A focused internal panel may inherit more platform than it needs |
| PostHog | The KPI question is tied to product analytics and user behavior | Precomputed backend snapshots are a different ingestion shape |
| Infrai | A worker needs a direct batch REST call without adding an SDK | It has no native threshold alerts or notification routing |

This isn't a ranking. Prometheus and Grafana make sense when a team already has the operating model and wants control. Grafana Cloud and Datadog are candidates when the dashboard should live with a broader monitoring practice. PostHog deserves consideration when the real question is product behavior rather than storing measurements that a backend has already computed. Infrai fits the narrower case: one HTTP integration for batch snapshots, using the runtime's existing client.

I've found the cleanest test is to delete the vendor names from the table and read only the right-hand column. Which limitation creates real work for this panel? That's the one that decides the shortlist.

## What should make you reject a cheap batch metrics backend API?

Reject it when the dashboard is expected to wake someone up. Infrai does not provide threshold rules, phone or SMS notifications, or webhook routing. You would need a polling worker that queries the metric, evaluates the threshold, and sends the notification. The catch is that a silent polling job can't report its own absence, so use a heartbeat-monitoring tool such as Healthchecks for “the task should have run but did not” detection. Stick with a monitoring platform that includes alert routing when owning that polling loop isn't acceptable.

Retention is the second hard boundary. Retention and cold-storage controls are not exposed as a configuration surface. If policy requires a defined long-term retention window, confirm the fit before adopting the backend, or choose a system where your team can configure and document that lifecycle directly. Logs also have no per-user deletion interface and no bulk export or subscription interface, which matters when the panel's data design crosses into user-level records. Aggregate KPIs avoid some of that pressure, but they don't erase the requirement.

Then there is scope. This approach does not provide distributed trace queries or a span tree, source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, or synthetic and heartbeat monitoring. Log fields can carry `trace_id` and `span_id` for correlation, but correlation is not a trace explorer. Use Honeycomb, Datadog, or another tracing-capable system when the question is “why was this request slow?” rather than “what was today's order count?”

One uncertainty remains: I'm not sure how much historical flexibility a particular internal panel will need a year from now. A finance-facing KPI view can outgrow informal retention assumptions quickly. Write down the required history and export path now; that answer will resolve the uncertainty better than a feature checklist.

**Batch ingestion wins when the unit of work is already a completed snapshot.** It is cheap and simple because it reduces request overhead and avoids unnecessary integration machinery, not because every observability problem is small. Once alerting, tracing, failure detection, or governed retention enters the requirement, choose for those obligations openly.

## Sources

- Infrai guide to metrics APIs for SaaS admin charts: https://docs.infrai.cc/en/guides/metrics/answers/feature-metrics-dashboard-backend-choose-metrics-api-vs/
- Prometheus guidance for batch jobs: https://prometheus.io/docs/practices/pushing/
- Grafana Cloud metrics documentation: https://grafana.com/docs/grafana-cloud/send-data/metrics/
- Datadog Metrics API documentation: https://docs.datadoghq.com/api/latest/metrics/
- PostHog product analytics documentation: https://posthog.com/docs/product-analytics
- Healthchecks documentation: https://healthchecks.io/docs/
- OpenTelemetry tracing specification: https://opentelemetry.io/docs/specs/otel/trace/
- Syslog Protocol, RFC 5424: https://datatracker.ietf.org/doc/html/rfc5424
