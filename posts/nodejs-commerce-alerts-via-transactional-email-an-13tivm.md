# Nodejs Commerce Alerts via Transactional Email and SMS (Polling Without Webhooks)

TL;DR: Treat a generated e-commerce report as one traceable notification, then let a scheduled worker poll its email and SMS receipts from durable storage. Record submission, receipt age, and the final reason separately. Retry an uncertain operation with the same logical key; stop retrying a permanent outcome. This gives operators useful evidence without running an inbound webhook receiver.

The important design choice is the evidence model. A successful API call means “submitted,” not “delivered,” and an email delivery receipt does not mean the attachment was read. If those three facts share one `sent` flag, every dashboard and retry decision built on top of it will lie.

Start with the operator's screen, not the transport SDK.

## Draw the dashboard before writing the sender

Picture the flow in words: order batch closes; a report renderer writes one immutable attachment; an email worker submits it; a polling worker checks the remote receipt; a policy may create an SMS alert; both channels end in an auditable outcome. The report identifier connects the line. The email and SMS identifiers branch from it.

The “before” dashboard has a single counter called `send_errors`. It cannot distinguish a renderer backlog from a rejected address or a late status lookup. The “after” dashboard needs a small set of signals with different owners: oldest render job age, oldest unsubmitted email age, unresolved receipt age, terminal failures by reason, and SMS escalations by policy reason. Consider a batch that creates 50 report notices, submits 48 emails, and leaves two render jobs waiting. A transport-wide delivery percentage hides the local bottleneck. Stage age exposes it immediately, while the report ID lets an operator follow one notice without searching by recipient data.

That split is practical. A growing render age points at local compute or storage. A growing receipt age says the external submission happened but knowledge of its outcome is stale. Terminal recipient failures belong in a correction workflow, not an infrastructure retry loop.

Use logs to connect those metrics. Each log event should carry a notification ID, report ID, channel, operation, attempt number, and normalized outcome. Do not log the attachment body, the full recipient address, or an SMS body. The useful question is “where did notification `n_42` stop progressing?” A recipient's private data rarely helps answer it.

Keep three clocks. `createdAt` measures total business delay. `submittedAt` starts receipt age. `nextCheckAt` schedules work. One timestamp cannot serve all three purposes.

## How should Nodejs event notifications poll transactional email and SMS status?

The poller should promise eventual reconciliation within a declared business deadline, not instant delivery knowledge. It claims due records atomically, asks the configured transport for current status, stores the observation, and schedules another check only for a nonterminal state or a retryable lookup error.

No guesswork.

The trade-off is visible in the operating model:

| Mechanism | Knowledge arrives | Application must operate | Main limitation |
| --- | --- | --- | --- |
| Scheduled polling | At the next due check | Outbound jobs, leases, and per-record schedules | Repeated reads and delayed status |
| Signed webhooks | After the callback reaches the receiver | Public ingress, verification, replay defense, and durable intake | Callback delivery and receiver availability |

A five-minute cron tick does not imply every record must be checked every five minutes. `nextCheckAt` is per notification. That distinction controls load and gives an operator a concrete explanation for why a receipt has not been queried yet. Add a small random offset if many reports become due at the same boundary; the goal is to avoid a synchronized burst, not to change the deadline.

Submission and polling also need separate retry budgets. A submission timeout is ambiguous: the remote side may have accepted the email even though the response was lost. Reuse a stable logical key when the transport supports idempotent submission, and reconcile before creating a second send. A failed status lookup never justifies resubmitting the attachment. It changes only when the next lookup runs.

Delivery vocabularies differ, so normalize them at the adapter boundary. The application needs a conservative internal set such as `pending`, `delivered`, `temporary_failure`, `permanent_failure`, and `unknown`. Preserve the raw value for diagnosis, but drive policy from the normalized value. An unfamiliar value maps to `unknown` and raises a compatibility signal. It must not silently become `delivered`.

## Make one worker copyable and boring

The following TypeScript focuses on the scheduling contract. Storage must implement `claimDue` as an atomic lease or equivalent compare-and-set operation, so overlapping cron processes cannot own the same transition.

```ts
type ReceiptState =
  | "pending"
  | "delivered"
  | "temporary_failure"
  | "permanent_failure"
  | "unknown";

type DueReceipt = {
  notificationId: string;
  reportId: string;
  channel: "email" | "sms";
  remoteId: string;
  checkAttempt: number;
  deadlineAt: Date;
};

interface ReceiptTransport {
  lookup(remoteId: string): Promise<{
    state: ReceiptState;
    rawState: string;
  }>;
}

interface ReceiptStore {
  claimDue(now: Date, limit: number): Promise<DueReceipt[]>;
  observe(input: {
    notificationId: string;
    state: ReceiptState;
    rawState: string;
    observedAt: Date;
    nextCheckAt?: Date;
  }): Promise<void>;
  markForReview(notificationId: string, reason: string): Promise<void>;
}

const delaysMs = [60_000, 120_000, 300_000, 600_000, 1_200_000] as const;

function nextDelay(attempt: number): number {
  return delaysMs[Math.min(attempt, delaysMs.length - 1)];
}

export async function reconcileReceipts(
  store: ReceiptStore,
  transportFor: (channel: DueReceipt["channel"]) => ReceiptTransport,
  now = new Date(),
): Promise<void> {
  const due = await store.claimDue(now, 50);

  for (const receipt of due) {
    if (now >= receipt.deadlineAt) {
      await store.markForReview(receipt.notificationId, "receipt_deadline_exceeded");
      continue;
    }

    try {
      const result = await transportFor(receipt.channel).lookup(receipt.remoteId);
      const terminal =
        result.state === "delivered" || result.state === "permanent_failure";

      await store.observe({
        notificationId: receipt.notificationId,
        state: result.state,
        rawState: result.rawState,
        observedAt: now,
        nextCheckAt: terminal
          ? undefined
          : new Date(now.getTime() + nextDelay(receipt.checkAttempt)),
      });
    } catch (error) {
      const reason = error instanceof Error ? error.name : "UnknownLookupError";
      await store.observe({
        notificationId: receipt.notificationId,
        state: "unknown",
        rawState: reason,
        observedAt: now,
        nextCheckAt: new Date(now.getTime() + nextDelay(receipt.checkAttempt)),
      });
    }
  }
}
```

The delays are explicit: 1, 2, 5, 10, then 20 minutes. They are an example operating policy, not a claim about a transport. Replace them with intervals that meet the report's deadline and permitted request volume. The hard stop is `deadlineAt`; without it, an unresolved receipt can consume polling work forever.

There is intentionally no email submission in this worker. Mixing submission and reconciliation makes the most dangerous retry path easy to trigger: a temporary lookup failure can accidentally send a second attachment. Keep generation, submission, reconciliation, and escalation as separate operations even if one process hosts all four.

For SMS, reuse the receipt machinery but not the email outcome. An SMS alert is a new policy decision with its own logical identity. A sensible e-commerce rule might create it only when the report is time-sensitive and email has reached a terminal failure or crossed its business deadline. The text should point the recipient toward a secure first-party destination rather than copying report data into the message.

## Test the timeline, not only the happy response

Adapter tests prove that raw transport statuses map correctly. Workflow tests should prove time and ownership. Freeze the clock, insert a due receipt, run the worker twice, and verify that the first lease prevents the second run from claiming it. Then advance time across each retry boundary and confirm that a terminal observation removes `nextCheckAt`.

Include the awkward cases: the process stops after a remote lookup but before the local write; a lease expires; the adapter returns a new raw status; the deadline arrives during repeated lookup failures; and an email failure creates exactly one SMS policy record. These tests expose integration effort far better than a successful sandbox send.

Deployment deserves a canary with a non-sensitive attachment. Watch queue age and normalized outcomes during rollout. Alert when the oldest eligible notification breaches its business deadline, when claimed work stops completing, or when unknown statuses appear. A single temporary lookup error is expected queue work. Paging on each occurrence trains operators to ignore the system.

For US and EU deployments, decide where recipient identifiers, report objects, notification records, and logs are allowed to reside before choosing an adapter. Region is a data-flow constraint. Separate regional workers can emit the same metric names and obey the same state contract while keeping storage and credentials within their intended boundaries.

## Why doesn't a delivered receipt close the report workflow?

Because transport evidence and user evidence answer different questions. A delivery state can report what the transport observed. It cannot establish that a person read the email, understood the report, or opened the attachment.

Open tracking does not repair that gap. Apple's Mail Privacy Protection can download remote content in the background and prevents senders from learning accurate details about Mail activity. Treating a tracking-pixel request as a human open would corrupt both operational alerts and product analytics. If the business needs proof that an authorized customer accessed a report, record a consented, first-party view or download event at the protected report surface. Keep that event out of the delivery state.

Email authentication belongs on another plane as well. DMARC defines domain policy, identifier alignment, and reporting around email authentication. It does not supply a per-recipient read receipt. Monitor authentication reports for domain-level health while the notification ledger tracks individual submissions and receipts.

This separation produces cleaner incident language. “Email receipt age is above the report deadline” is actionable. “Customers did not engage” is a product question. They should never share an alert merely because both began with the same email.

## Choose the boundary that keeps evidence intact

Polling fits when the transport exposes a status lookup, delayed knowledge is acceptable, and the team prefers scheduled outbound work over operating a public callback receiver. It costs recurring requests and can learn about changes only on the next check. Webhooks fit tighter notification latency, but add signature verification, replay handling, public ingress, and durable intake before acknowledgement. Neither mechanism proves readership.

This design is not suitable when status must change in near real time, the transport has no lookup operation, or the allowed request volume cannot support the required cadence. In those cases, use a signed webhook receiver backed by durable intake, or choose a transport boundary that supplies the evidence the application needs. The reverse limitation matters too: a webhook-only design is a poor fit when the deployment cannot expose and defend public ingress.

Judge integration effort by the evidence you can preserve: a stable remote identifier, documented status meanings, enough status retention for the polling window, scoped credentials, atomic queue ownership, and a testable distinction between temporary and permanent outcomes. SDK method count is a weak proxy.

The durable rule is short: generate the report once, submit it under a stable logical identity, reconcile receipts without resending, and escalate through an explicit policy. Then show each boundary in logs and metrics. Operators can see delay, retries remain controlled, and a transport change does not rewrite the meaning of delivery.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, “Use Mail Privacy Protection on iPhone”: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
