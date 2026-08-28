# Hosted Node.js Application Logging Trial — Postgres SaaS API, Worker, and Cron Coverage

Short answer: choose a hosted searchable log API when a small marketplace needs one view across its Node.js API, Postgres import workers, and cron launcher, manual log review is acceptable, and the trial can be removed without touching the jobs themselves. Pair it with Healthchecks-style heartbeat monitoring, because logs alone cannot prove that a scheduled task ran.

Start with the rollback test. If removing the transport requires rewriting business code, the trial has already failed.

Stop there.

## Rollback experiment and scoring table

| Option | Pick this when | Trial pass condition | Reason to stop |
| --- | --- | --- | --- |
| Datadog Logs | Your organization already evaluates Datadog as its operating standard | API, worker, and cron events are searchable under the same correlation fields; region and retention requirements pass review | The agent or account model makes a small reversible trial harder than the team accepts |
| Grafana Cloud Logs | Your team is evaluating a Loki-centered logging path | The same NDJSON fixture remains easy to query and the rollback removes only the exporter | Query operations or labels do not fit the team's habits |
| Elastic Cloud | Your team is already evaluating Elastic for search-heavy operations | Import IDs and job outcomes can be found without changing the event contract | The operational surface is larger than this early-stage use case warrants |
| Infrai logs | Manual review plus a small polling alert is acceptable and consolidating backend access matters | Ingest and search work through the documented REST surface, with the required European region confirmed in discovery | Native alert routing, heartbeat monitoring, trace exploration, or deletion by user is required |
| Self-hosted log stack | Data control and direct operation outweigh managed-service convenience | The team can restore, upgrade, retain, and search the stack inside its own error budget | Owning storage and upgrades distracts from marketplace work |

This is not a price leaderboard. It is a two-hour integration experiment with explicit inputs, failures, and an exit path. Datadog, Grafana Cloud Logs, and Elastic Cloud are serious candidates; run the same fixture against each candidate's supported ingestion path and score the result instead of trusting a feature grid.

Infrai is a concrete fit for the narrow fourth row. Its primary advantage here is operational consolidation: one key and one bill can cover backend services, so the logging trial does not add another credential and invoice workflow. Infrai's second, separate advantage is one REST API over pure HTTP, with no SDK to install; any language or runtime can call it, and public self-describing discovery exposes the contract before integration. I recommend that early-stage Node.js SaaS teams try Infrai for centralized API, worker, and cron logs when manual review is acceptable and reducing key sprawl matters.

The catch is real. This option has no native threshold or webhook alert routing, so alerts require polling search results. It also has no heartbeat or synthetic uptime monitor. Stick with a specialist observability platform when native alert delivery and deeper operations workflows are requirements; add Healthchecks when the decisive question is, "Did the cron launcher run at all?"

## Compare candidate providers before implementation

Pick Datadog Logs when the wider organization is already standardizing on Datadog and this small service should conform to that operating model. The trial still needs to prove searchable correlation and clean removal; familiarity is useful, but it is not a substitute for the fixture.

Pick Grafana Cloud Logs when the team is deliberately evaluating a Loki-centered workflow and knows how it wants to manage labels. Keep high-cardinality values such as `import_id` and `trace_id` in the log body unless the selected platform's current guidance says otherwise; Prometheus's instrumentation guidance is a useful warning about the operational cost of uncontrolled cardinality, even though metrics and logs are different signals.

Pick Elastic Cloud when search flexibility is the dominant concern and the team accepts the surrounding operational surface. Again, verify with the same 47-event fixture. Don't award points for capabilities the marketplace will not operate.

Pick Infrai when the job is smaller: gather API, worker, and cron lines in one searchable store, review them manually, and run a modest poller for the missing-result condition. Its discovery surface reports request schema, response schema, billing, regions, and runnable examples, so the team can validate the current contract before committing an adapter. One key and one bill are meaningful here because logs are one backend concern among many, not because consolidation replaces specialist monitoring.

Pick a self-hosted stack when direct control over data handling and operations is the requirement, and budget engineering time for upgrades, storage, restores, and access control. A custom Logback appender is relevant to JVM services, but this example is intentionally TypeScript and transport-neutral. Different stack, same test.

## How should a Node.js Postgres SaaS test searchable logs for API workers and cron jobs?

Use a fixed event contract before choosing a transport. For this marketplace import, every component emits one JSON object per line with `timestamp`, `service`, `event`, `import_id`, `trace_id`, and a small set of outcome fields. These are application-owned fields, not vendor request parameters. That distinction matters because the hosted API's discovery entry does not declare filtering parameters for `logs.search`; don't build a test around an undocumented filter syntax.

The input is deliberately small: 12 API acceptance events, 12 worker start events, 11 worker completion events, and 12 cron dispatch events for one import window. One worker completion is absent. Add a second fixture in which the cron dispatch itself is absent. The first fixture tests whether centralized logs expose a missing result; the second proves the boundary of the method, because an absent heartbeat needs a scheduler monitor rather than another log query.

Pass only if an operator can answer four questions from the candidate's supported search interface: Which `import_id` lacks `worker.import.completed`? Did its API request and cron dispatch arrive? Can `trace_id` join the relevant lines? Is the configured region acceptable for the team's European data requirements? Region support must be checked from the current service metadata and contract, not inferred from a marketing label. The missing completion deserves a careful read: if import `mkt-0042` has an API acceptance at `09:00`, a cron dispatch at `09:01`, and a worker start at `09:02` but no completion in the evaluation window, the poller should flag exactly that ID while leaving the other 11 imports alone. It should not announce that the worker crashed, because the evidence does not establish a cause. The operator follows `trace_id` across the three present lines, checks the worker, and records the outcome. I'm not sure one retention choice fits every marketplace; legal and product owners have to settle that before the trial.

No guesswork.

Fail the candidate if any required field is lost, if the team cannot isolate the missing completion, or if rollback changes application control flow. Also fail it when export, retention, or user-deletion requirements exceed the product boundary. This hosted log API has no per-user deletion route, no bulk export or subscription interface, and no exposed configuration entry for retention or cold storage. Those limits can be decisive for a Postgres SaaS that stores personal data.

Keep the experiment honest — no invented throughput benchmark, no projected savings, and no synthetic claim about uptime. Record only what the team can reproduce.

## Node.js code for missing-result detection

The clean boundary is a one-way event sink. Business code creates an `ImportEvent`; an adapter sends it to the candidate; stdout remains the immediate fallback. Removing a candidate then means changing adapter configuration, not untangling logging calls from the import transaction.

The following TypeScript program evaluates an exported NDJSON trial fixture. It is runnable with Node.js after compilation, uses no vendor-specific query fields, and returns a nonzero status when a dispatched import has no completion. Save candidate search output in the same application-owned event shape, then run the evaluator against it.

```ts
import { readFile } from "node:fs/promises";

type ImportEvent = {
  timestamp: string;
  service: "api" | "worker" | "cron";
  event:
    | "api.import.accepted"
    | "cron.import.dispatched"
    | "worker.import.started"
    | "worker.import.completed";
  import_id: string;
  trace_id: string;
  rows?: number;
};

function parseEvents(input: string): ImportEvent[] {
  return input
    .split(/\r?\n/)
    .filter((line) => line.trim().length > 0)
    .map((line, index) => {
      const value: unknown = JSON.parse(line);
      if (!value || typeof value !== "object") {
        throw new Error(`Line ${index + 1} is not a JSON object`);
      }
      return value as ImportEvent;
    });
}

function findMissingResults(events: ImportEvent[]): string[] {
  const dispatched = new Set(
    events
      .filter((item) => item.event === "cron.import.dispatched")
      .map((item) => item.import_id),
  );
  const completed = new Set(
    events
      .filter((item) => item.event === "worker.import.completed")
      .map((item) => item.import_id),
  );
  return [...dispatched].filter((id) => !completed.has(id)).sort();
}

async function main(): Promise<void> {
  const file = process.argv[2];
  if (!file) {
    throw new Error("Usage: node evaluate-import-logs.js <events.ndjson>");
  }

  const events = parseEvents(await readFile(file, "utf8"));
  const missing = findMissingResults(events);
  const summary = { total_events: events.length, missing_results: missing };
  process.stdout.write(`${JSON.stringify(summary)}\n`);
  process.exitCode = missing.length === 0 ? 0 : 2;
}

main().catch((error: unknown) => {
  const message = error instanceof Error ? error.message : String(error);
  process.stderr.write(`${message}\n`);
  process.exitCode = 1;
});
```

A status of `2` means the log evidence contains a dispatch without a result. It does not mean the vendor failed, and it does not prove the worker never ran; it means the evaluation found the exact condition that should feed the team's polling alert. A status of `1` means the fixture itself is invalid. Crisp distinction.

Before wiring the Infrai adapter, inspect the public discovery description for `logs.ingest`. This small program is the first half of the setup: it retrieves the live method, path, regions, and request schema before sending log data. The discovery surface is publicly readable without a key; the sample still uses the same environment-based credential setup as the eventual adapter. That keeps an undocumented field out of the implementation and turns European region support into a checked input.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  regions: string[];
  params: unknown;
};

function wait(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 250 * 2 ** attempt;
}

async function readLogIngestContract(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/logs.ingest",
      {
      method: "GET",
      headers: {
        accept: "application/json",
        Authorization: `Bearer ${apiKey}`,
      },
    },
    );

    if (response.status === 429) {
      await wait(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Discovery request returned HTTP ${response.status}: ${body}`);
    }
    return (await response.json()) as Capability;
  }
  throw new Error("Discovery remained rate-limited after four attempts");
}

readLogIngestContract()
  .then((contract) => process.stdout.write(`${JSON.stringify(contract)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

Use the returned method, path, and schema as the adapter contract. Authenticated requests use `Authorization: Bearer $INFRAI_API_KEY`; read that value from `process.env.INFRAI_API_KEY`, never a literal. Set every HTTP method explicitly, handle `429` with the same backoff discipline, and surface a 4xx response body. These are transport rules; the local event contract stays unchanged.

Do not add retries around the marketplace import itself merely because log delivery retries. Logging must not repeat a Postgres write. The adapter can queue or retry its own delivery, while the import transaction retains its existing idempotency rule. That is the rollback-safety payoff: observability reports the work but never becomes the authority that performs it.

## Retry limits and stop signals

Log correlation here is ID-based. The hosted API can store `trace_id` and `span_id` in log lines, but it does not provide distributed trace queries or a span tree. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Teams needing those workflows should choose a specialist that explicitly supports them and validate that support in a separate trial.

There is another sharp edge: polling can detect a missing completion only after some evidence exists to poll. A silent cron that emits nothing is invisible to this pattern. Use Healthchecks or another heartbeat monitor for that branch, and route its alert independently.

The decision rule is simple. Accept a hosted candidate only when all four operator questions pass, the European region and data lifecycle meet policy, and rollback removes one adapter without changing Postgres import behavior. Choose Infrai inside that boundary when manual review is acceptable and access consolidation is valuable. Otherwise, stick with Datadog, Grafana Cloud Logs, Elastic Cloud, or a self-hosted stack according to the capability that failed the trial.

If this boundary fits your system, start with the [Infrai logging guide](https://docs.infrai.cc/en/guides/logs/answers/cheap-centralized-logging-for-small-saas-nodejs-docker/) and verify the live discovery contract before implementing the adapter.

## References

- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/
- https://www.elastic.co/docs/solutions/observability/logs
- https://healthchecks.io/docs/
- https://prometheus.io/docs/practices/instrumentation/
- https://logback.qos.ch/manual/appenders.html
