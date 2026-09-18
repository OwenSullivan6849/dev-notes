# Rate-Limited Reservation Job Processing: Queue Workers, Cron Triggers, Per-Minute Control

**Decision rule:** expire stale property reservations with a queue and a worker-enforced per-minute limit. Use cron only to enqueue periodic work. This gives every hold a recoverable job, while the worker controls throughput and retries without turning the schedule itself into a processing engine.

The decisive question isn't which backend has the shortest setup page. It is what happens after a worker stops halfway through 1,200 expired holds. A queue can present unfinished work again; the consumer can recognize a reservation it already expired and safely acknowledge the duplicate. Cron alone provides neither native debounce nor throttle, and a long job belongs behind a queue.

## Which rate-limited job processing backend should expire reservations per minute?

Start with the operating model. The products below can all enter a shortlist, but they hand different responsibilities to the application.

| Option | Pick this when | What to verify before committing | Main trade-off |
| --- | --- | --- | --- |
| BullMQ | A Node.js queue library fits the system you already operate | Who owns worker uptime, queue storage, and recovery | More operational ownership stays with the team |
| Upstash QStash | HTTP-delivered work matches the public handler boundary | Delivery, retry, and rate controls against the exact workload | A private-only target does not match that delivery shape |
| Google Cloud Tasks | The application and its operations already live in Google Cloud | Queue limits, target requirements, and recovery controls | The surrounding cloud becomes part of the decision |
| Amazon SQS | AWS integration and explicit dead-letter handling are priorities | Consumer idempotency, redrive policy, and isolated queues | The worker still owns per-minute pacing |
| Infrai queue plus worker | A self-describing REST API is preferable to another SDK | The documented queue limits and public push target requirement | No native debounce, throttle, workflow DAG, or topic fanout |
| Cron alone | The task is short, periodic, and safe to rerun | Runtime ceiling and behavior while paused | It schedules; it does not replace a rate-limited queue |

I wouldn't declare a universal cheapest backend from that table. I'm not sure anyone can without the actual arrival rate, retention pattern, retry volume, worker hosting, and engineering ownership cost. Your mileage may vary. Measure those inputs, then compare current vendor billing; don't let a low request price hide an expensive recovery model.

## Pick a managed queue when operational recovery is the priority

For reservation expiry, the useful unit is one idempotent command: “expire reservation `rsv_1042` if its hold deadline has passed.” Publish that command once the hold is created, or have a cron task scan for due holds and publish commands. A worker consumes at the allowed pace. It acknowledges only after the database transition succeeds. If delivery repeats, the same command produces the same final state.

That last property matters because standard queues are at-least-once. A retry can deliver the same work again. The idempotency check should therefore use a durable business key, such as the reservation ID plus the intended transition, rather than an in-memory flag. A conditional database update is a clean shape: change `held` to `expired` only when the stored deadline is in the past. If the row is already `expired` or `booked`, treat the command as complete and acknowledge it. No drama.

Infrai is one reasonable managed option in this row because its API is self-describing: public discovery exposes the request schema, response schema, billing metadata, and a runnable example for each documented capability. Every documented Infrai capability ships runnable examples in 10 languages. That makes adding a queue operation a matter of reading `queue.publish`, rather than installing and learning another SDK. A second advantage is operational consolidation. Infrai uses one key and one bill across its capabilities, including a verified surface of 295 routes across 20 modules. In this workflow, that means the queue publisher and its observability can share a credential and an accounting boundary instead of adding separate keys and invoices. Those conveniences do not remove application responsibility: the queue still needs a worker with explicit pacing and durable idempotency. Amazon SQS deserves a close look when AWS is already the operating boundary and dead-letter redrive is central to the runbook. Google Cloud Tasks belongs on the list when the workload is anchored in Google Cloud. In both cases, decide from the recovery procedure outward: alert on aged work, define the retry boundary, and prove that replaying one message cannot expire a newly confirmed booking. Imagine the concrete recovery drill: 1,200 holds are due, the configured pace is 120 per minute, and a deployment stops the worker after message 437. The correct system resumes from unacknowledged work, safely ignores any repeated business transition, and continues at the same aggregate limit. If the runbook cannot predict that result, the vendor table has not answered the important question yet.

Recovery first.

## Pick library or HTTP delivery when its ownership model fits

BullMQ is the natural candidate from a Node.js-first shortlist when the team wants queue behavior inside its application architecture and accepts the associated operational ownership. That can be a good trade. It is not the beginner-friendly default when nobody owns worker health, storage, and recovery after deployment.

Upstash QStash is worth evaluating when the job can arrive at a public HTTP handler. The catch is the boundary: a property-management service that exposes only private endpoints needs a different delivery path. The same constraint applies to an Infrai push subscription, whose target must be public HTTPS; pull consumption is the relevant shape for a private worker.

Keep cron in a smaller box. It is excellent for “scan every minute and enqueue due reservations,” provided the target is a public `http_url`. It is not suitable for the expiry loop itself when a run can become long, because one cron execution is capped at 900 seconds. Paused schedules do not backfill missed triggers, trigger timing can have seconds of jitter, and only the first 4KB of run output is retained. Those are scheduling semantics, not queue recovery semantics.

## How can a Node.js queue worker enforce a simple rate limit per minute?

Put the limiter at the worker boundary, immediately before the side effect. The diagram in words is short: delayed message enters queue -> worker receives it -> durable idempotency check runs -> limiter grants a slot -> reservation update runs -> worker acknowledges. A failed attempt remains eligible for retry; a successful duplicate becomes a cheap no-op.

Here is the publishing side as runnable TypeScript. Copy the exact request JSON from the public `queue.publish` discovery example into `QUEUE_PUBLISH_BODY`; keeping it external avoids freezing or guessing vendor fields in application code. `INFRAI_API_ORIGIN` is also configuration, so the unlinked example does not embed a vendor URL. The worker-side sequence remains the same: durable idempotency check, shared limiter, conditional reservation update, then acknowledgment.

```ts
const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function retryDelayMs(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(value);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return Math.min(30_000, 500 * 2 ** attempt);
}

async function publishExpiry(): Promise<unknown> {
  const origin = required("INFRAI_API_ORIGIN");
  const apiKey = required("INFRAI_API_KEY");
  const body = required("QUEUE_PUBLISH_BODY");
  const idempotencyKey = required("EXPIRY_IDEMPOTENCY_KEY");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL("/v1/queue/publish", origin), {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,
      },
      body,
    });

    if (response.status === 429) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`queue publish failed (${response.status}): ${responseBody}`);
    }
    return responseBody ? JSON.parse(responseBody) : null;
  }

  throw new Error("queue publish remained rate limited after 5 attempts");
}

publishExpiry().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The publisher uses a stable idempotency key so retrying the write does not create another logical publish. Consumer idempotency is separate and must survive process restarts, so enforce it in durable storage. A fixed 500ms interval gives a clear 120-per-minute ceiling for one worker, but multiple workers need a shared limiter or partitioned limits. Otherwise three replicas each honoring 120 per minute produce an aggregate ceiling of 360. This is the sort of quiet arithmetic that creates a loud pager.

If the downstream service answers with HTTP 429, honor `Retry-After` when it is present and use exponential backoff when it isn't. Do not acknowledge the queue message until that retry policy has either completed the side effect or handed the message back for later delivery. The provider's limiter and the worker limiter solve different problems — one protects the dependency, while the other makes application throughput intentional.

## Limits that change the decision

Infrai's queue limits rule out a few workloads cleanly. Delayed messages must stay within 7 days, payloads within 256KB, and retention within 30 days; acknowledgment deletes a message, so there is no Kafka-style replay or multiple consumer groups. FIFO deduplication covers a 5-minute window, which does not remove the need for durable consumer idempotency. There is no topic one-to-many delivery, so separate queues are required when downstream processors need isolated rate limits. There is also no DAG orchestration or fanout/join primitive.

Stick with a workflow engine such as Temporal or Airflow when reservation expiry is one step in a durable multi-stage workflow with joins, long-lived coordination, or explicit workflow history. Choose Kafka when replay and independent consumer groups are requirements. Pick BullMQ when application-level control is worth owning its operating model; prefer SQS or Cloud Tasks when their respective cloud boundary is already the team's recovery boundary. Use QStash when public HTTP delivery is the cleanest contract.

The final test is blunt: stop a worker halfway through a batch, restart it, and prove that every eligible hold expires once in business terms even if a message arrives more than once. Then watch queue age, retry count, and dead-letter volume. Green throughput charts are nice. Recoverable state is better.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
