# Node.js Regional Error Log Triage for US and EU SaaS Backends

Short answer: poll each log region with its own committed cursor, turn error records into stable failure fingerprints, and post one aggregated Slack notification per fingerprint instead of forwarding every line.

The alert is only half the system. A useful watcher must also avoid losing an event between fetching and checkpointing, avoid paging the same failure hundreds of times, and report when polling itself goes quiet. Keep those three jobs separate.

The mental model is short: **before**, each log line becomes a message; **after**, the watcher reads a regional stream, classifies failures, groups related records, applies a notification window, delivers a summary, and only then advances durable state. That ordering makes retries boring. Boring is good.

## How should Node.js detect backend failures while polling an error logs API?

Treat every region as an independent ordered stream. A US cursor must never stand in for an EU cursor, because one region can be delayed or unavailable while the other continues. The logs API URL in the example is configuration, not an invented universal route. Its response contract is equally explicit: an ordered `events` array and a `nextCursor` value. Adapt that small boundary to the API you actually operate.

Do not equate an HTTP request failure with an application failure. The first means the watcher could not inspect a stream; the second is evidence found inside that stream. They need different alerts, owners, and deduplication keys. Mixing them produces a nasty loop: a failed poll looks like a backend incident, the poll runs again, and the notification channel fills with copies while no new evidence has been read.

Diagram in words: regional logs API -> fetch page -> validate records -> select error events -> compute fingerprint -> update group -> notify Slack -> commit regional cursor. A separate arrow runs from poll health -> stale-region alert. There is no shortcut from fetch failure to cursor commit.

| Concern | State key | Failure policy |
| --- | --- | --- |
| Regional progress | Region plus cursor | Retry without advancing |
| Alert grouping | Region plus fingerprint | Aggregate within the window |
| Watcher health | Region plus last successful poll | Alert from a separate monitor |

Fingerprinting matters because raw messages often contain request IDs, timestamps, and numeric identifiers. Grouping systems use event attributes and configurable fingerprints to decide which events belong to one issue; the same principle works in a small polling worker. Start with stable fields such as service and error code, then normalize only the known variable fragments in the message. Aggressive normalization can merge unrelated failures. Conservative normalization can create extra alerts. Your mileage may vary, so test the rule against captured, sanitized fixtures before deployment.

## The copyable polling worker

This TypeScript example defines the boundary instead of guessing it. `LOGS_API_URL_US` and `LOGS_API_URL_EU` must each contain the real query endpoint for that deployment. The code expects the endpoint to accept `cursor` and `limit` query parameters; if your provider uses another contract, change `fetchPage` and leave the grouping and commit sequence alone.

The state store is intentionally an interface. Back it with a transactional database or another durable store in production. An in-memory implementation is useful for tests, but it cannot preserve cursors or cooldowns across a restart.

```ts
type Region = "us" | "eu";

type LogEvent = {
  id: string;
  timestamp: string;
  level: "debug" | "info" | "warn" | "error";
  service: string;
  message: string;
  errorCode?: string;
};

type LogPage = {
  events: LogEvent[];
  nextCursor: string;
};

type GroupState = {
  count: number;
  firstSeen: string;
  lastSeen: string;
  lastNotifiedAt?: string;
};

type Checkpoint = {
  cursor?: string;
  groups: Record<string, GroupState>;
};

interface StateStore {
  load(region: Region): Promise<Checkpoint>;
  commit(region: Region, checkpoint: Checkpoint): Promise<void>;
}

const regionUrls: Record<Region, string> = {
  us: requiredEnv("LOGS_API_URL_US"),
  eu: requiredEnv("LOGS_API_URL_EU"),
};

const slackWebhookUrl = requiredEnv("SLACK_WEBHOOK_URL");
const notificationWindowMs = 10 * 60 * 1000;

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

function fingerprint(event: LogEvent): string {
  const normalizedMessage = event.message
    .replace(/[0-9a-f]{8}-[0-9a-f-]{27,}/gi, "{uuid}")
    .replace(/\b\d{4,}\b/g, "{number}");

  return [event.service, event.errorCode ?? "no-code", normalizedMessage].join("|");
}

async function fetchPage(
  region: Region,
  cursor: string | undefined,
): Promise<LogPage> {
  const url = new URL(regionUrls[region]);
  if (cursor) url.searchParams.set("cursor", cursor);
  url.searchParams.set("limit", "200");

  const response = await fetch(url, {
    method: "GET",
    headers: { accept: "application/json" },
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    throw new Error(`Log poll failed with status ${response.status}`);
  }

  return (await response.json()) as LogPage;
}

async function sendSlack(text: string): Promise<void> {
  const response = await fetch(slackWebhookUrl, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text }),
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    throw new Error(`Slack delivery failed with status ${response.status}`);
  }
}

function shouldNotify(group: GroupState, now: Date): boolean {
  if (!group.lastNotifiedAt) return true;
  return now.getTime() - Date.parse(group.lastNotifiedAt) >= notificationWindowMs;
}

async function pollRegion(region: Region, store: StateStore): Promise<void> {
  const checkpoint = await store.load(region);
  const page = await fetchPage(region, checkpoint.cursor);
  const now = new Date();

  for (const event of page.events) {
    if (event.level !== "error") continue;

    const key = fingerprint(event);
    const group = checkpoint.groups[key] ?? {
      count: 0,
      firstSeen: event.timestamp,
      lastSeen: event.timestamp,
    };

    group.count += 1;
    group.lastSeen = event.timestamp;
    checkpoint.groups[key] = group;

    if (shouldNotify(group, now)) {
      await sendSlack(
        `[${region.toUpperCase()}] ${event.service}: ${event.errorCode ?? "error"}\n` +
          `${event.message}\nOccurrences: ${group.count}; first seen: ${group.firstSeen}`,
      );
      group.lastNotifiedAt = now.toISOString();
      group.count = 0;
    }
  }

  checkpoint.cursor = page.nextCursor;
  await store.commit(region, checkpoint);
}

async function runOnce(store: StateStore): Promise<void> {
  const results = await Promise.allSettled([
    pollRegion("us", store),
    pollRegion("eu", store),
  ]);

  results.forEach((result, index) => {
    if (result.status === "rejected") {
      const region: Region = index === 0 ? "us" : "eu";
      process.stderr.write(`Poll unhealthy for ${region}: ${String(result.reason)}\n`);
    }
  });
}
```

Notice the commit point. Walk one page through the ugly timing: the US poll reads cursor `c41`, receives three matching errors and `c42`, sends a summary, and then the process exits before `commit` finishes. On restart, `c41` is still durable, so the worker reads that page again and may repeat the summary. Reverse the two writes and the opposite failure appears — if `c42` is committed before the webhook call and the process exits at that instant, those three errors are now behind the durable cursor and no notification is sent. Exactly-once delivery is not promised here. This is the part teams tend to discover during a real interruption, when nobody wants to debate write ordering. An outbox changes the shape: store the alert intent and the new cursor in one transaction, deliver pending intents separately, and identify each intent from region, cursor, and fingerprint so a retry can be recognized. It adds storage, a delivery loop, and cleanup work, but it is the right trade when duplicate or missing pages carry a serious operational cost.

Commit last.

Also validate the JSON at the adapter boundary in real code. A cast tells TypeScript what a developer hopes the server returned; it does not inspect runtime data. Reject a page with a missing cursor, quarantine malformed events, and emit a metric for both. Fast failure beats silently committing past data the worker never understood.

## From noisy lines to actionable incidents

An error-level filter is a starting point, not a production detection policy. Some libraries log expected client mistakes as errors. Other backends record a failed dependency call at `warn`. Build the classifier from explicit signals: service, environment, error code, exception type, deployment version, and a small allowlist of known messages. Keep the original event ID in internal state so an operator can retrieve context without placing an entire stack trace, email address, or access token in Slack.

The before/after is crisp. Before: 347 repeated lines can become 347 interruptions. After: one fingerprint opens an alert, the worker increments its occurrence count during a ten-minute window, and the next notification reports the aggregate. The number here describes the example, not a benchmark. Use fixtures with one isolated error, a burst of duplicates, two similar messages with different codes, an out-of-order timestamp, an empty page, and a malformed response. I use a synthetic `429` response in the delivery test to prove that the cursor stays put; it is a test input, not a claim about a production incident.

Redact before egress.

That rule matters in both US and EU deployments, but data residency is not the whole privacy story. Slack receives whatever the watcher sends. Keep message construction narrow, set retention deliberately, and maintain a deletion path for personal data. GDPR Article 17 defines a right to erasure and also lists conditions and exceptions, so the correct retention and deletion design depends on the data and legal basis. I'm not sure a generic log scrubber can settle that question for a particular company; a data inventory and legal review can.

Operationally, record poll duration, events examined, failures classified, groups suppressed, notifications attempted, notification failures, cursor age, and last successful poll time per region. Alert on cursor age from a separate monitor. Otherwise the watcher can stop, produce no application alerts, and look perfectly calm.

## What are the limits of this design?

Polling is a good fit when the logs API is the available integration point, detection latency can be roughly the polling interval plus API delay, and query volume stays within the API's operating limits. The catch is that shorter intervals increase request volume while longer intervals delay detection. Pages also need a stable cursor or an equally clear ordering contract. Without one, records arriving late around a timestamp boundary can be skipped or read twice.

This design is not suitable when every second matters, when the source cannot provide durable pagination, or when alert volume is high enough that one worker becomes a queue. In those cases, stick with a streaming or push-based ingestion path and put durable buffering between detection and delivery. Keep the same classifier and fingerprint tests; replace the transport.

A Slack webhook is convenient for a team notification, but it isn't an incident state machine. It does not supply ownership, acknowledgement, escalation, or resolution semantics in this example. Route high-severity failures into the organization's established incident process, and reserve chat messages for concise context. Don't put secrets or raw personal data in them.

There is another tradeoff in the checkpoint order. Commit after notification favors avoiding lost alerts but permits duplicates. Commit before notification favors fewer duplicates but can lose an alert. The outbox pattern resolves much of that tension at the cost of another durable component and cleanup policy. Choose explicitly. Then test the crash points on both sides of every write.

## References

- Sentry, "Event Grouping": https://docs.sentry.io/concepts/data-management/event-grouping/
- GDPR, "Article 17: Right to erasure": https://gdpr-info.eu/art-17-gdpr/

## Further reading

The two primary references above are the useful next stops: the event-grouping documentation explains why fingerprints affect issue boundaries, while GDPR Article 17 supplies the legal text behind erasure requirements and exceptions.
