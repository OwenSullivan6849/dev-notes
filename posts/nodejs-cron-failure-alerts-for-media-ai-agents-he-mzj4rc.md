# Nodejs Cron Failure Alerts for Media AI Agents: Healthchecks or Error Tracking?

For a media AI agent, healthchecks and error tracking cover different cron job failure alerts: a healthcheck catches a missed report, while error tracking explains code that ran and failed. A process can return exit code `0` and still produce nothing worth publishing. That operational constraint changes the monitoring choice.

Short answer: use an external healthcheck to detect a missed start or completion deadline, and use error tracking to explain failures inside a run. For a scheduled Nodejs agent that generates media, keep both signals, join them with one run ID, and page only on the condition that best represents reader impact. Measure latency and cost as run attributes, not as extra reasons to wake someone up.

The choice isn't really healthchecks versus error tracking. It is absence detection versus execution evidence. One owns the clock. The other owns the crash context.

## What can a green cron dashboard fail to prove?

The before model looks comforting: cron starts the Nodejs process, the process calls an AI model, and an error tracker reports thrown exceptions. No exception appears, so the dashboard stays green. Yet that green state says nothing about a schedule entry that never fired. It also says nothing about a process that started and stopped making progress without throwing. Silence has two meanings: success, or missing evidence.

The after model is a diagram in words: **schedule expectation -> start deadline -> completion deadline -> useful outcome**. An observer outside the worker owns the first two deadlines. The worker emits execution details. A durable result records what the media system actually accepted, such as the count of publishable items. These are related signals, but they aren't interchangeable.

That last state matters for AI work. A completed request may still yield an empty result, a rejected editorial decision, or an output that the publishing pipeline declines. None of those outcomes has to be a software exception. Treating process completion as business success would make the alert quiet and the morning queue empty.

So give every scheduled run three independent questions:

1. Did it start within the expected window?
2. Did it finish before its completion deadline?
3. Did it produce an acceptable durable outcome?

Short and sharp.

A healthcheck answers the first two by comparing a remote expectation with received signals. Error tracking answers a narrower but richer question: what exception, stack, and execution context appeared after code began running? The outcome record answers the third. Logs and metrics then help explain timing, volume, and spend without pretending that every unusual value is an incident.

## How should Nodejs cron failure alerts combine healthchecks and error tracking?

Start with the failure contract, not a dashboard. Write down the expected schedule, allowed start delay, maximum run duration, and definition of a useful result. The clock must live outside the Nodejs process. If the worker owns its own deadline, a missing worker also removes the evidence needed to detect that it is missing.

For a media agent loop, use one `runId` across the start heartbeat, model-call records, error event, durable output, and completion heartbeat. The alerting path can then group two symptoms from one run instead of opening two incidents. A missed completion should be the primary symptom; a captured exception is context attached to it. If an exception occurs and the worker also reports an explicit failed completion, deduplication should keep the on-call channel to one actionable page.

Europe and the US don't require different detection mechanics. They do require deliberate ownership. Store schedule expectations in UTC, display the relevant local time to the responder, and document what a local-time publishing rule does during daylight-saving changes. A job tied to local midnight needs an explicit policy for a repeated or skipped civil hour. The remote expectation must implement the same policy as the scheduler.

Keep heartbeat payloads sparse — a pseudonymous job name, `runId`, state, and timestamp are enough for correlation. Model prompts, generated copy, customer data, and stack traces belong only in systems approved for that data class and region. I'm not sure what residency boundary your contracts impose; an engineering diagram can't decide that. The answer comes from the actual data inventory, processor terms, and internal policy.

Here is the decision split:

| Evidence | Detects well | Cannot establish alone |
| --- | --- | --- |
| External schedule healthcheck | No start, no completion, late completion | Why executed code failed |
| Error tracking | Thrown and reported execution failures | That a process never existed |
| Run metrics | Latency, call counts, token counts, cost inputs | Whether one unusual value deserves a page |
| Durable outcome | Accepted media output and idempotency state | Whether the next scheduled run will happen |

Don't turn that table into four alert channels. It is four evidence types for one run.

## Instrument the agent loop once, then derive the signals

The following TypeScript wrapper uses generic ports rather than a vendor endpoint. The monitoring adapter can deliver events over HTTP, write them to a queue, or persist them locally. The pricing adapter is deliberately outside this example because model prices are external configuration that can change; the loop records actual usage and receives the computed cost from that boundary.

```ts
type RunState = "started" | "completed" | "failed";

type RunEvent = {
  job: string;
  runId: string;
  state: RunState;
  at: string;
  elapsedMs?: number;
  modelCalls?: number;
  inputTokens?: number;
  outputTokens?: number;
  costUsd?: number;
  acceptedItems?: number;
  errorName?: string;
};

type AgentResult = {
  modelCalls: number;
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
  acceptedItems: number;
};

interface RunObserver {
  emit(event: RunEvent): Promise<void>;
  capture(error: unknown, context: { job: string; runId: string }): Promise<void>;
}

interface MediaAgent {
  generate(runId: string): Promise<AgentResult>;
}

async function runScheduledAgent(
  agent: MediaAgent,
  observer: RunObserver,
  now: () => Date = () => new Date(),
): Promise<void> {
  const job = "regional-media-agent";
  const runId = crypto.randomUUID();
  const startedAt = now();

  await observer.emit({
    job,
    runId,
    state: "started",
    at: startedAt.toISOString(),
  });

  try {
    const result = await agent.generate(runId);
    const finishedAt = now();

    await observer.emit({
      job,
      runId,
      state: "completed",
      at: finishedAt.toISOString(),
      elapsedMs: finishedAt.getTime() - startedAt.getTime(),
      ...result,
    });
  } catch (error) {
    await observer.capture(error, { job, runId });
    await observer.emit({
      job,
      runId,
      state: "failed",
      at: now().toISOString(),
      errorName: error instanceof Error ? error.name : "UnknownError",
    });
    throw error;
  }
}
```

The external monitor still needs the cron expectation and completion deadline; code inside the worker cannot prove that the worker never started. The wrapper's job is correlation. It makes a run visible from start to durable outcome and gives latency and cost the same join key.

Test the boundaries before deployment. Trigger a normal run, prevent the scheduler from launching a test run, stop a test worker after its start event, make the agent adapter throw, and return a completed result with `acceptedItems: 0`. Those five cases should produce five distinguishable records. Only the cases that violate the written failure contract should page. The zero-item case may be valid on a quiet news cycle, so its policy needs a business rule rather than a hard-coded assumption.

There is one nasty edge. The media output can commit successfully while the completion event fails to reach the observer. The external clock will then report a missed completion even though the work exists. Don't automatically rerun non-idempotent publishing work from that alert. Persist `runId` with the output, make retries idempotent, and let the responder reconcile the durable result before retrying. A cautious false positive is annoying; duplicate publication is reader-visible damage.

## Signal quality beats a larger pile of alerts

Latency and cost are useful dimensions, but raw thresholds create noise when story complexity and model-call count vary by run. Record total elapsed time, model-call count, input and output usage, computed cost, accepted-item count, and region. Then compare like with like: the same job, schedule class, and region over a representative window. A daily investigative report can surface drift without paging on every outlier.

Percentiles are often more useful than an average because a small number of slow runs can disappear inside the mean. web.dev assesses Core Web Vitals at the 75th percentile; that is evidence for the aggregation pattern, not a ready-made threshold for agent jobs. Your agent's alert boundary must come from its publishing deadline and observed distribution. Copy the habit, not the number.

I'm not sure p75 is the right operational percentile for every media loop. A job with a hard edition cutoff may care about the single late run, while a high-frequency enrichment loop may tolerate several slow executions. Resolve that uncertainty by replaying the proposed rule against historical run records and counting how many pages would have been actionable.

Use alert budgets as a review tool. For each proposed rule, record the condition, intended responder action, and suppression key. If nobody can name the immediate action, make it a dashboard or ticket signal. If two rules lead to the same action for the same `runId`, group them. This is where the before/after becomes tangible: before, five telemetry streams can produce five pages; after, one missed deadline opens the incident and the other streams explain it.

## When is one signal enough, and what is the catch?

Error tracking alone is suitable when the work is request-driven, every meaningful failure necessarily executes instrumented code, and a missing schedule is outside the requirement. A healthcheck alone can be suitable for a simple scheduled transfer when the only operational question is whether it completed on time and the durable result already contains enough diagnosis. Neither choice is a moral victory. Scope decides.

The limitation of the hybrid is extra event handling. It is not suitable for a team that has no way to correlate or deduplicate its events, because adding a second alert source can reduce signal quality. Fix the run identity and routing contract first. Also avoid heartbeat paging for event-driven workers with no promised execution cadence; queue age or oldest-unprocessed-item time is the meaningful absence signal there.

The catch with an external completion deadline is that it observes evidence delivery, not business truth. Network delay can make a healthy run look late, and a completion signal can arrive before an unsafe output is durably committed if the instrumentation is placed too early. Send completion only after the accepted result is durable, allow a grace period justified by the schedule, and keep the output idempotent.

For a scheduled media AI agent, the clean design is modest: an independent clock catches silence, error context explains executed failures, and one run record carries latency, usage, cost, and accepted output. Page on broken commitments. Study the rest.

## Sources

- https://web.dev/articles/vitals
