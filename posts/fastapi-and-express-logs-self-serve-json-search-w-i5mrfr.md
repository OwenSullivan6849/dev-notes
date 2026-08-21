# FastAPI and Express Logs: Self-Serve JSON Search with Cost Attribution

Short answer: for a small logistics team, start with newline-delimited JSON on standard output, a managed collector path, and one searchable dashboard; preserve an unsampled cost event for every AI agent run, keyed by tenant, shipment, model, and correlation ID. This is the least complex setup that makes production logs self-serve without turning application logging into a second platform.

| Pick | Use it when | Cost-attribution fit | Main trade-off |
| --- | --- | --- | --- |
| Hosted log backend | The team wants search, dashboards, retention, and access control with minimal operations | Strong if numeric cost fields and stable dimensions are searchable | Less control over residency, retention behavior, and query economics |
| Cloud-native logging | The app already runs in one cloud and its identity and regional controls are acceptable | Strong when billing exports and application fields can share tenant identifiers | Migration and cross-cloud analysis can become awkward |
| Self-managed collector and store | Residency, custom retention, or predictable infrastructure ownership dominates | Strong, but the team owns schema, indexes, upgrades, and recovery | Highest operational load |

The table is the decision. The implementation below is the guardrail: emit one durable shape from FastAPI or Express, then let the transport and storage choice change independently.

## What must structured production JSON logs expose for FastAPI or Express teams?

Choose on operational ownership first, then verify five capabilities with a one-hour proof: ingest newline-delimited JSON without rewriting fields; filter by `tenant_id` and `agent_run_id`; aggregate `latency_ms` and `cost_usd_micros`; restrict access by environment or tenant; and pin storage to the required US or EU region. A polished dashboard is useful. Correct attribution is the gate.

For a logistics agent, one customer action may call a routing model, inspect shipment context, retry a tool, and produce a final answer. A single request total hides the expensive step. Record a summary event for the run and a child event for each model or tool step. Both carry the same correlation ID. The run event answers, "What did this shipment cost?" The child events answer, "Why?"

Consider a hypothetical run with correlation ID `req_7f2`, shipment `shp_1842`, and two model steps. The first step classifies a delivery exception; the second drafts the next action after a tool lookup. A request-level event can report 840 ms and a combined cost, yet it cannot reveal whether classification, generation, or the tool dominated. Two child events can. If the second step retries, give each attempt its own event ID while keeping the run ID stable, then let the terminal summary include the final token totals and attempt count. Now an engineer can search one run, a team lead can aggregate by tenant, and finance can reconcile final usage without treating duplicated delivery as duplicated spend. The point is not to prescribe those sample values. It is to make the grain of each record explicit before a backend's defaults quietly choose it for you.

Search first.

Don't turn every value into a label. High-cardinality values such as `shipment_id` and `agent_run_id` belong in searchable log fields; a bounded field such as `environment` is a better metric label. Prometheus naming guidance also recommends a base unit and one logical unit per metric. That makes a metric such as `agent_run_duration_seconds` easier to aggregate than a mixture of milliseconds and seconds.

Keep it boring.

The same event contract can cross the Python/JavaScript boundary. FastAPI can serialize the shape in Python, while an Express service can use the TypeScript definition below; the field names and semantics matter more than matching logger libraries. Use UTC timestamps, integer minor units for money, explicit units in field names, and a schema version. Those choices prevent quiet errors such as adding dollars to microdollars or comparing milliseconds with seconds.

## Hosted ingestion reduces the operational surface

A hosted backend is the easiest default for a team that doesn't want to run indexes, storage tiers, dashboard authentication, and upgrades. Pick it when engineers need to search a `429` response by correlation ID during an incident and then group the same events by tenant without asking a platform specialist to prepare a query. Test that workflow with representative cardinality before signing a long retention commitment.

The catch is control. This option is not suitable when policy requires a storage topology the service cannot provide, when tenant isolation needs controls beyond its access model, or when variable ingestion and query volume cannot be budgeted. In those cases, use the cloud-native or self-managed path. I'm not sure which retention window will fit an unfamiliar workload, and a brochure can't settle it; replay a scrubbed day of representative events, measure indexed volume, and test the queries the on-call engineer will actually run. Your mileage may vary.

## Regional identity narrows the storage boundary

Cloud-native logging removes another integration when compute identity, encryption policy, audit access, and regional placement already live in one provider boundary. It is a practical choice for a US-only or EU-only deployment where the existing cloud controls satisfy the team's review. The app should still write portable JSON to standard output — coupling belongs in deployment configuration, not every logging call.

Stick with this path when one cloud is an intentional constraint. Don't choose it merely because ingestion is already enabled: first prove that a developer can move from `agent_run_id` to all child steps, chart p95 run latency, sum attributed cost, and exclude test traffic without privileged console knowledge. If a second region or cloud is likely soon, estimate the cost of duplicated dashboards, access rules, and exports before committing.

## Self-managed storage transfers recovery ownership

A self-managed collector and store fits teams with strict data-location rules, unusual retention tiers, or an existing observability platform staffed to operate the added workload. It can keep the application contract clean: services emit JSON, a collector batches and routes it, and the store indexes only the fields needed for investigation and aggregation.

But it isn't the easy choice for most small teams. Someone must own backpressure, disk capacity, index mapping, upgrades, restore tests, query limits, and authentication. A dashboard that works on launch day does not prove recoverability. If there is no named owner or tested recovery objective, pick a managed path and keep the event schema portable.

## Implement one cost ledger beside diagnostic logs

Diagnostic logs and cost records have different failure budgets. Debug events may be sampled or retained briefly. A cost ledger should emit exactly one summary per completed or failed run and should not be sampled, because missing expensive runs biases both totals and averages. OpenTelemetry's sampling documentation distinguishes head decisions made before a trace completes from tail decisions made after more trace information is available; whichever trace policy you use, keep the cost ledger independent from sampled diagnostic detail.

Here is a focused TypeScript contract. The model rate is configuration, not a hard-coded market claim, and the calculation stores microdollars as an integer. The event contains no prompt, address, parcel description, or model response. Those payloads create privacy and search problems and are unnecessary for attribution.

```ts
import { randomUUID } from "node:crypto";

type AgentOutcome = "ok" | "failed";

type AgentRunLog = {
  schema_version: 1;
  event_name: "agent_run_completed";
  timestamp: string;
  environment: "production" | "staging";
  service: string;
  tenant_id: string;
  shipment_id: string;
  agent_run_id: string;
  correlation_id: string;
  model: string;
  outcome: AgentOutcome;
  input_tokens: number;
  output_tokens: number;
  tool_calls: number;
  latency_ms: number;
  cost_usd_micros: number;
  error_code?: string;
};

type ModelUsage = {
  model: string;
  inputTokens: number;
  outputTokens: number;
};

type ModelRate = {
  inputUsdPerMillionTokens: number;
  outputUsdPerMillionTokens: number;
};

function calculateCostUsdMicros(usage: ModelUsage, rate: ModelRate): number {
  return Math.round(
    usage.inputTokens * rate.inputUsdPerMillionTokens +
      usage.outputTokens * rate.outputUsdPerMillionTokens,
  );
}

function writeAgentRun(event: AgentRunLog): void {
  process.stdout.write(`${JSON.stringify(event)}\n`);
}

async function observeAgentRun(
  tenantId: string,
  shipmentId: string,
  correlationId: string,
  rate: ModelRate,
  execute: () => Promise<{ usage: ModelUsage; toolCalls: number }>,
): Promise<void> {
  const started = process.hrtime.bigint();
  const agentRunId = randomUUID();

  try {
    const result = await execute();
    writeAgentRun({
      schema_version: 1,
      event_name: "agent_run_completed",
      timestamp: new Date().toISOString(),
      environment: "production",
      service: "shipment-agent",
      tenant_id: tenantId,
      shipment_id: shipmentId,
      agent_run_id: agentRunId,
      correlation_id: correlationId,
      model: result.usage.model,
      outcome: "ok",
      input_tokens: result.usage.inputTokens,
      output_tokens: result.usage.outputTokens,
      tool_calls: result.toolCalls,
      latency_ms: Number(process.hrtime.bigint() - started) / 1_000_000,
      cost_usd_micros: calculateCostUsdMicros(result.usage, rate),
    });
  } catch (error) {
    writeAgentRun({
      schema_version: 1,
      event_name: "agent_run_completed",
      timestamp: new Date().toISOString(),
      environment: "production",
      service: "shipment-agent",
      tenant_id: tenantId,
      shipment_id: shipmentId,
      agent_run_id: agentRunId,
      correlation_id: correlationId,
      model: "unknown",
      outcome: "failed",
      input_tokens: 0,
      output_tokens: 0,
      tool_calls: 0,
      latency_ms: Number(process.hrtime.bigint() - started) / 1_000_000,
      cost_usd_micros: 0,
      error_code: error instanceof Error ? error.name : "UnknownError",
    });
    throw error;
  }
}
```

There is a deliberate limitation in that failure branch: if a provider consumed tokens before throwing, zero is not a trustworthy final cost. Production code should take usage from the provider response when available, or reconcile the provisional event with an authoritative usage record. Do not guess. Add `usage_status: "final" | "pending"` if reconciliation is asynchronous, then make dashboards sum only final records.

Ship the event through a collector with bounded buffering, retry limits, and a dead-letter destination. Alert on missing ledger events by comparing the number of agent starts with terminal outcomes over a window; alert separately on collector drops. For the dashboard, begin with four panels: run count, p95 `latency_ms`, total `cost_usd_micros`, and cost per successful run. Add filters for tenant, environment, model, outcome, and time. This compact view answers the operational question before anyone builds a wall of charts.

That part matters.

Test the pipeline as a data contract. A deployment check should emit one synthetic run, search it by correlation ID, verify that numeric fields remain numeric, confirm that a restricted user cannot see another tenant, and confirm deletion after the configured retention window. Load testing should include realistic long identifiers and cardinality, not 100 copies of one tidy event. A schema compatibility test can then reject a rename from `cost_usd_micros` to `cost` before it silently empties the dashboard.

## Know the limits before calling the dashboard self-serve

Structured logs are not a billing system. They are excellent for investigation and near-real-time attribution, but retries, delayed usage, duplicate delivery, and retention can make them diverge from an authoritative invoice. Reconcile totals outside the log store and attach a stable event ID if financial reporting depends on them.

Self-serve also has a boundary. If tenant fields contain personal or shipment-sensitive data, broad search access is the wrong goal; minimize fields, redact before emission, and enforce scoped access. If the team needs causal timing across many services, add traces and correlate them with the same ID. If it needs low-cardinality service health trends, add metrics with consistent names and base units. Logs, traces, and metrics should share identifiers, but each signal has a different job.

The final choice is conditional: use hosted ingestion for the smallest operations footprint, cloud-native logging for an intentional single-cloud boundary, and self-managed storage only when control is worth owning recovery and capacity. Keep the JSON contract portable, keep the cost ledger unsampled, and make the proof-of-choice query follow one shipment from request to final attributed cost.

## References

- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
