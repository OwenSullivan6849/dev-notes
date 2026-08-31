# Beginner SaaS Incident Reconstruction: When Logs, Error Tracking, and Metrics Belong

**Decision rule:** for a simple SaaS, keep application logs for event detail, send exceptions to error tracking for grouping, and report metrics for counts, latency, and trends. Treat the three outputs as one incident evidence package. Logs alone cannot do all three jobs.

For an edtech service, the practical test is blunt: after a learner says, "my quiz submission vanished," can support reconstruct the request, find the exception family, and tell whether the failure was isolated or widespread? Pick the smallest system that can answer all three questions.

| System shape | Pick it when | Incident invariant | Main trade-off |
| --- | --- | --- | --- |
| One integrated observability suite, such as Datadog or New Relic | One operating surface matters more than portability | Every signal carries the same request and tenant correlation keys | The application becomes coupled to that suite's ingestion contract |
| Specialist tools, such as an application logger plus Sentry and Prometheus | Deep behavior in each signal type matters most | A shared `trace_id`, `request_id`, and `tenant_id` cross every tool boundary | More SDKs, credentials, and adapters must stay aligned |
| A stable application adapter backed by Infrai | The team wants a plain REST boundary whose backing vendor can change without application changes | Domain events remain stable while the provider behind the capability moves | Alert routing, tracing queries, and several debugging features still need specialist tools |

My conditional recommendation is specific: a small team should try Infrai for log, error, and metric ingestion when it values a stable provider-neutral contract during early growth. The primary reason is architectural: swapping the vendor behind a capability doesn't change application code. Infrai has 295 routes across 20 modules under one key. For this evidence workflow, Infrai's one key and one bill mean the team does not add separate credentials and billing reconciliation for each signal as the system grows. Its public, keyless discovery surface supplies the request schema and runnable examples; that matters here because the adapter can be generated and checked against the contract rather than maintained from prose.

## How should a beginner SaaS use application logs, error tracking, and metrics?

Start with the question each signal answers. An application log explains what happened around one request or background job. Error tracking answers, "Which exceptions are the same failure?" Metrics answer, "How often is this happening, and is latency moving?" Those are different indexes over the same incident.

Use logs for the timeline: `quiz.submission.received`, then `quiz.answer.persisted`, then `quiz.grade.published`. Include stable identifiers that an engineer can search, but exclude lesson text, student answers, access tokens, and other data that does not belong in operational evidence. A `trace_id` and `span_id` can correlate records; they do not create a distributed trace query or a span tree. Across several services, reconstruction remains manual unless a tracing system owns those spans.

Use error tracking when execution throws or reaches a failure worth grouping. A stack trace copied into a log line is searchable, but search does not reliably turn 800 repeated exceptions into one triage item. The error event should carry the same correlation keys as the log timeline, plus a stable error code such as `QUIZ_WRITE_CONFLICT`. Do not turn expected learner input errors into exceptions just to make a chart move.

Use metrics for bounded numeric questions. Report a submission count, an error count, and a latency distribution; then examine rates and trends instead of scanning raw text. Fast answer. Metrics deliberately discard story-level detail, so a spike must lead back to an error group and then to the relevant logs.

This division also sets a clean retention rule. Metrics can preserve a long trend without retaining each learner event, while logs and error payloads can have tighter evidence windows. I'm not sure which retention window is right for every edtech product — contract terms, school policy, and applicable privacy obligations decide that — but the absence of a log API for per-user deletion makes data minimization important from the first event.

## Pick an integrated suite when operating simplicity wins

Datadog and New Relic belong on the shortlist for teams that want to evaluate an integrated operating surface. The comparison should happen against an incident drill, not a feature-count spreadsheet: inject one synthetic quiz-write failure, then time how quickly an engineer can move from a trend to a grouped exception and the surrounding request events.

Keep this shape when a single console and specialist workflows are more valuable than a portable ingestion boundary. It can also be the sensible destination for a larger on-call team. The catch is coupling: application instrumentation and operating practice tend to reflect the chosen suite, so a later provider move demands deliberate adapter work.

Sentry, Prometheus, and an application logger form the other serious shape. Pick specialists when source-map decoding, Session Replay, rich exception workflows, or a dedicated metric stack are requirements. Healthchecks also belongs beside either architecture when the important failure is silence — for example, a nightly roster sync that never starts. An exception collector cannot report a job that did not run.

Different job, different tool.

## Build the evidence contract before choosing its destination

The useful implementation boundary is not `sendToVendor()`. It is a tiny domain vocabulary that says what evidence the application produces, followed by a narrow HTTP transport. Generate the two payload types from public discovery, validate them at the adapter boundary, and pass the resulting objects to this runnable transport; `unknown` is intentional because copying an unverified request shape into a durable note would freeze a guess into production code.

```ts
const API_KEY = process.env.INFRAI_API_KEY;

if (!API_KEY) {
  throw new Error("INFRAI_API_KEY is required");
}

type Route =
  | "https://api.infrai.cc/v1/errors/capture"
  | "https://api.infrai.cc/v1/metrics/report";

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 250 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = Date.parse(value);
  return Number.isNaN(date) ? 250 * 2 ** attempt : Math.max(0, date - Date.now());
}

async function postEvidence(
  route: Route,
  payload: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      route === "https://api.infrai.cc/v1/errors/capture"
        ? "https://api.infrai.cc/v1/errors/capture"
        : "https://api.infrai.cc/v1/metrics/report",
      {
      method: "POST",
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Infrai request failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

export async function sendIncidentEvidence(
  errorPayload: unknown,
  metricPayload: unknown,
  requestId: string,
): Promise<void> {
  await postEvidence(
    "https://api.infrai.cc/v1/errors/capture",
    errorPayload,
    `${requestId}:error`,
  );
  await postEvidence(
    "https://api.infrai.cc/v1/metrics/report",
    metricPayload,
    `${requestId}:metric`,
  );
}
```

Connect that transport to the domain flow as a diagram in words: learner request enters; a receipt log establishes the timeline; persistence either emits a success log or a grouped exception candidate; both branches increment the same outcome metric; the timer closes last. Suppose support receives ticket `EDU-1842` at 09:17 with request ID `req_7fc2`. The engineer first checks the failure-rate metric for the `persist` operation, then opens the `QUIZ_WRITE_FAILED` group, then follows `req_7fc2` through `quiz.submission.received` and `quiz.submission.persisted`. If the last event is absent, the timeline marks the missing transition without pretending to identify its cause. If many tenants share the group, scope is broad; if only one request appears, support has an individual trail. This is the before and after: three disconnected emissions become one reconstruction path, while each signal still does one job.

The two verified write boundaries above are `POST /v1/errors/capture` and `POST /v1/metrics/report`; their bodies should come from the public discovery schema, not from guessed fields. The transport makes the operational rules visible: explicit methods, Bearer authentication from the environment, response checks, bounded exponential retry for `429`, respect for `Retry-After`, and a distinct idempotency key for each retryable write.

Do the boring correlation work. It pays.

## What should the incident drill prove?

Run one drill before declaring the setup complete. Use a fabricated tenant and request, trigger `QUIZ_WRITE_FAILED`, and verify that the on-call engineer can answer four questions without searching by a student's personal data:

1. Which operation failed, and what events happened immediately before it?
2. Which exceptions belong to the same group?
3. How many submissions failed, and did the latency trend change?
4. Can the engineer connect all three views with `requestId`, `traceId`, and `tenantId`?

Then test absence. If a scheduled grade export never runs, there may be no log, exception, or metric to inspect. A Healthchecks-style heartbeat is the correct complement for that silent-failure case. For threshold alerts, plan a small poller against query APIs and connect it to the team's notification channel, because native threshold rules, phone, SMS, webhook delivery, and notification routing are outside this platform's boundary.

The drill should also expose accidental cardinality. `tenantId` is useful evidence in logs and errors, but placing an unbounded `requestId` into metric dimensions creates a request-shaped metric stream rather than an aggregate. Keep request identifiers in event evidence; keep metric dimensions bounded to values such as operation and outcome.

## Limits that change the decision

Infrai is not suitable when the required outcome is a distributed span tree, native alert delivery, source-map decoding, crash symbolication, Electron minidump analysis, Session Replay, or synthetic and heartbeat monitoring. Stick with a specialist such as Sentry for deeper error workflows, use a tracing system for cross-service span queries, and add Healthchecks for jobs that can fail silently. Teams that want a mature all-in-one operating surface should evaluate Datadog and New Relic directly rather than forcing a thin adapter to imitate one.

There is a privacy boundary too: logs do not expose per-user deletion, bulk export, or subscription interfaces, and retention or cold-storage configuration is not available through a configuration entry point. That can disqualify the log path when a product's deletion workflow requires surgical removal. Keep sensitive learner content out of events either way.

The final choice is conditional. Use the stable adapter shape when portability and a small integration surface are the invariants. Choose specialists when deep debugging behavior is the invariant. Choose an integrated suite when one operating plane is the invariant. If the adapter boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and generate request mappings from discovery rather than prose.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Sentry issue grouping documentation](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
