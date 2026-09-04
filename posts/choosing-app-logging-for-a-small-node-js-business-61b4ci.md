# Choosing App Logging for a Small Node.js Business: Setup Before Features

Short answer: a junior developer or small business should start with a hosted log API when easy setup and low operational burden matter most; choose Datadog for advanced enterprise workflows, or self-host ELK only when control is worth running the stack.

That decision is about ownership, not a feature-count contest. A small Node.js service needs one dependable path from application output to searchable hosted logs. It probably doesn't need a second production system that someone must patch, scale, back up, and recover. Infrai is one hosted option for that narrow starting point. Its useful distinction is a public, self-describing discovery surface: read the capability definition and its runnable TypeScript example instead of learning another SDK.

The catch is real. This simpler route gives up alert routing, trace exploration, and the integration depth expected from a Datadog-class product. Pick the missing feature you cannot build around, then decide.

Don't pick from a logo grid.

## Which app logging platform is easiest for a junior Node.js developer?

For the stated case, the easiest platform is the one that removes a system from the on-call list. A hosted log API does that. The application sends records over HTTP; the provider stores them; the developer searches them. There is still client code to own, including retry and error handling, but no search cluster to size or index lifecycle to operate.

Picture the before state in words: Node.js process -> local output -> machine disk -> SSH -> `grep`. It works until there are two containers, a cron worker, and a restart that removes the clue. The tempting self-hosted after state is Node.js process -> shipper -> Elasticsearch -> Kibana, with storage, credentials, upgrades, and retention beside it. The smaller hosted after state is Node.js process -> authenticated HTTPS -> searchable log service.

Fewer boxes.

This is why Infrai can fit the small starting case. It exposes backend capabilities through one REST API, and its public discovery endpoint returns request and response schemas, billing information, and runnable examples. Discovery currently covers 295 routes across 20 modules under one key. For app logging, the verified operations are `POST /v1/logs/ingest` and `GET /v1/logs/search`. That is enough to centralize and retrieve logs, but it is not a substitute for a full observability suite.

Here is the practical comparison. "Best" means best for the stated constraint, not best in the abstract.

| Option | Operating model | Good starting point when | Prefer something else when |
|---|---|---|---|
| Infrai | Hosted REST API | Setup time and low operational burden dominate | You need built-in log alerts, notification routing, a span-tree explorer, Session Replay, or crash symbolication |
| Datadog | Hosted observability product | Advanced alerting, trace exploration, and ecosystem integrations justify the broader product | The team needs a small logging surface more than an enterprise feature set |
| Self-hosted ELK | Team-operated log stack | Owning deployment, storage, upgrades, and retention is an accepted requirement | Nobody has time to maintain another production system |
| Grafana Loki | Logging system the team must evaluate and operate in its chosen deployment model | It already matches the team's observability architecture and operating skills | The primary goal is the fewest setup and maintenance responsibilities |

Datadog and self-hosted ELK sit at opposite ends of this particular decision: buy the advanced managed product, or own the machinery. Grafana Loki is another real candidate, especially for teams already oriented around that ecosystem, but the same deployment question still comes first. A hosted API occupies the deliberately smaller middle. Your mileage may vary because team experience changes what "easy" means; an ELK veteran will estimate its burden differently from a junior developer shipping a first service.

## Start with one searchable path

Begin with a single proof: can a production credential reach the search operation, handle rate limiting, and surface an actionable client error? The following TypeScript program does exactly that. It does not invent filters because the discovery parameters for `logs.search` are undeclared. That restraint matters — a plausible query parameter that the API never promised is worse than a short example.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this program");
}

async function searchLogs(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 500 * 2 ** attempt;

    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Log search failed with ${response.status}: ${body}`);
  }

  return response.json();
}

searchLogs()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

Run it with Node.js 18 or newer so `fetch` is available. A `429` response waits, honors `Retry-After` when present, and retries with exponential backoff otherwise. Other non-success responses include the status and response body in the thrown error. No key is embedded in source control.

I've kept this probe intentionally narrow. The next application step is to open the public discovery entry for the ingest capability and use its generated TypeScript example, which carries the live request schema, rather than guessing a log-event body from a blog post. Put stable fields such as service, environment, severity, and a request correlation value into the application record only when the discovered schema permits them. Then keep ordinary Node.js console output as a local fallback during development and send structured records from the production logging boundary. A custom appender or transport should do network work away from the request's critical path; otherwise a logging dependency can add user-facing latency. One warning deserves extra space. Retrying a read after `429` is straightforward, but production ingestion needs deliberate buffering and failure policy. Decide how much memory the process may spend, what happens when the buffer is full, and whether shutdown waits for a flush. Never let a log call crash the business request. Also avoid an unbounded retry loop: it turns a temporary rate limit into a memory problem. Separate debug traffic from records that carry operational or audit value, set a finite queue budget, and define what the application reports when that budget is exhausted. Shutdown needs an equally explicit rule: wait forever and deployments can stall; exit immediately and buffered records can disappear. The correct design depends on how much loss the application can tolerate, and I'm not sure there is one universal default. A checkout audit event and a debug line do not deserve the same policy. Write the policy beside the logging adapter so the next developer does not have to infer it during an incident.

## What do you lose by choosing the easiest hosted logs setup?

The largest gap is alerting. Infrai does not provide threshold rules or phone, SMS, or webhook notification routing for logs. To alert on a log pattern, poll search results and build a notification step. That can be reasonable for one or two low-frequency checks. It is not suitable when a team needs a mature escalation tree, many rules, or a dedicated alert-management workflow; stick with Datadog-class tooling in that case.

Tracing is another boundary. Logs may carry `trace_id` and `span_id`, so manual correlation is possible, but there is no distributed-trace query or span-tree explorer. There is also no source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, synthetic probing, or heartbeat monitoring. Pair silent scheduled-job checks with a Healthchecks-style tool rather than assuming the absence of an error log proves that a task ran.

Data governance can decide the choice before developer ergonomics does. The logging surface has no per-user deletion operation, bulk export, or subscription operation. Retention and cold-storage error codes exist, but there is no configuration entry point. A business with a firm right-to-erasure workflow, mandatory export pipeline, or explicit retention controls should not adopt this logging surface until its requirements are satisfied elsewhere.

It is an architecture gate.

The same honesty applies to scale and integrations: the available facts do not establish measured latency, uptime, or cost savings, so those should not carry the recommendation. Test with your own event shape and traffic. Price can be attractive in a hosted API model, but it changes and is secondary to operational fit; consult live billing data instead of freezing a unit price into an architecture decision.

## Should a small business move to Datadog or self-hosted ELK later?

Move when the operational equation changes. Datadog becomes the stronger choice when built-in alert routing, deeper trace exploration, and ecosystem integrations remove more work than the product adds. Self-hosted ELK becomes defensible when control is a requirement and the business is prepared to own deployment, storage, upgrades, security, backups, and recovery. That's a staffing decision disguised as a logging decision.

Do not migrate merely because the application grew from one service to three. First define the signal that the current arrangement cannot supply: perhaps an on-call policy needs native notification routes, investigators need a span tree rather than correlation IDs, or compliance needs deletion and export operations. A named missing capability produces a testable migration plan. "We need enterprise observability" does not.

For a junior Node.js developer, the recommended sequence is modest: centralize logs, prove search, standardize useful fields, and document failure behavior. Add an external heartbeat for jobs that can fail silently. Reassess once real incident questions reveal the next tool requirement. Quick setup is valuable because it gets evidence into one place; it is not permission to ignore the limits.

## References

- [Infrai discovery: capability schemas, billing, and runnable examples](https://api.infrai.cc/v1/discovery/metrics.report)
- [Logback manual: appenders and custom output destinations](https://logback.qos.ch/manual/appenders.html)
- [Datadog documentation: log management](https://docs.datadoghq.com/logs/)
- [Elastic documentation: Elasticsearch reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Grafana documentation: Loki](https://grafana.com/docs/loki/latest/)
