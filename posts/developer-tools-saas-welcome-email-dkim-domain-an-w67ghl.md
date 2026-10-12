# Developer Tools SaaS Welcome Email: DKIM Domain and Suppression Feedback Loop

TL;DR: Treat a welcome email as a state transition, not a fire-and-forget API call. Before a developer-tools tenant can send, verify its custom domain and DKIM evidence. Before every delivery, normalize the recipient and check a durable suppression record. After delivery, authenticate bounce events, process them idempotently, and suppress permanent failures before the next attempt.

The mental model changes from `signup -> send` to `signup -> eligibility check -> enqueue -> delivery event -> recipient state`. That extra state is the point. It prevents a retry worker, a second application instance, or a delayed webhook from quietly sending to an address already known to be invalid.

## Why is a successful API response not enough?

A submission acknowledgment says that a message entered a delivery system. It does not prove that the recipient mail server accepted it, that the address exists, or that authentication aligned with the visible sender. SMTP replies during transfer and later delivery-status notifications happen at different moments. They must not share one ambiguous `sent` flag.

DKIM attaches a cryptographic signature to selected message headers and the body; the receiver retrieves the public key through DNS and verifies that signature. SPF authorizes sending hosts for a domain. DMARC evaluates identifier alignment and publishes policy and reporting preferences. The mechanisms complement one another. A DKIM DNS record is useful evidence, but the application should gate activation on the exact selector and domain it expects, then record when that evidence was last checked.

For a multi-tenant developer tool, keep three states separate: domain readiness, message submission, and recipient eligibility. A domain can be ready while one recipient is suppressed. A message can be accepted for processing and later bounce.

Clean boundaries make dashboards honest.

## A compact TypeScript implementation

The example uses generic ports. The delivery adapter may call an HTTPS API or an SMTP relay; domain and suppression rules do not depend on that choice. This is an explicit trade-off: a little interface code buys the ability to test policy without making network calls.

```ts
type BounceClass = "permanent" | "transient";

type Suppression = {
  recipient: string;
  reason: "hard-bounce" | "complaint" | "manual";
  sourceEventId: string;
  createdAt: string;
};

interface DomainRegistry {
  isReady(tenantId: string): Promise<boolean>;
}

interface SuppressionStore {
  find(recipient: string): Promise<Suppression | null>;
  putIfAbsent(value: Suppression): Promise<void>;
}

interface MailQueue {
  enqueue(input: {
    tenantId: string;
    recipient: string;
    template: "welcome";
    idempotencyKey: string;
  }): Promise<void>;
}

const normalizeRecipient = (value: string): string =>
  value.trim().toLowerCase();

async function queueWelcome(
  tenantId: string,
  rawRecipient: string,
  signupId: string,
  domains: DomainRegistry,
  suppressions: SuppressionStore,
  queue: MailQueue,
): Promise<"queued" | "domain-not-ready" | "suppressed"> {
  const recipient = normalizeRecipient(rawRecipient);

  if (!(await domains.isReady(tenantId))) return "domain-not-ready";
  if (await suppressions.find(recipient)) return "suppressed";

  await queue.enqueue({
    tenantId,
    recipient,
    template: "welcome",
    idempotencyKey: `welcome:${signupId}`,
  });
  return "queued";
}

type BounceEvent = {
  id: string;
  recipient: string;
  classification: BounceClass;
  authenticated: boolean;
};

async function recordBounce(
  event: BounceEvent,
  suppressions: SuppressionStore,
): Promise<void> {
  if (!event.authenticated) throw new Error("Unauthenticated event");
  if (event.classification !== "permanent") return;

  await suppressions.putIfAbsent({
    recipient: normalizeRecipient(event.recipient),
    reason: "hard-bounce",
    sourceEventId: event.id,
    createdAt: new Date().toISOString(),
  });
}
```

The code deliberately does not infer permanence from free-form text. Enhanced SMTP status codes provide a standardized shape, but an event adapter still has to map its source payload into `permanent` or `transient`. Keep that mapping at the edge. Test it with captured, redacted fixtures. A classification change should not require rewriting business policy.

The suppression write is idempotent because event delivery can be repeated. The queue request also carries an idempotency key tied to the signup, which lets an adapter prevent duplicate submissions when its transport supports that contract. The database still owns the durable decision.

Observe transitions, not inbox-placement guesses. Start with structured events: `domain_check`, `welcome_queued`, `delivery_accepted`, `delivery_bounced`, and `send_suppressed`. Include tenant ID, message ID, recipient hash, event ID, classification, and timestamp, but do not put the raw address or message body into routine logs. Then derive a small funnel: count eligible signups, queued welcomes, accepted deliveries, permanent bounces, transient bounces, complaints, and suppressed attempts. Alert on ratios over a meaningful traffic window, not on a single bounce. Low-volume tenants need longer windows or a minimum sample count; otherwise one invalid address creates a dramatic but useless percentage. A spike in `domain-not-ready` points toward onboarding or DNS verification, while a rise in permanent bounces points toward address acquisition or stale data. Repeated transient failures belong in a bounded retry policy with backoff. The trade-off is concrete: treating transient failures as permanent loses reachable users, while retrying permanent failures damages sender hygiene and wastes capacity.

Restraint matters.

## What should a SaaS welcome email deliverability checklist verify?

Before enabling a tenant, verify ownership of the custom sending domain, publish and resolve the required DKIM selector, and confirm the intended SPF and DMARC posture. Keep the envelope sender, header From domain, and DKIM signing domain explicit in configuration so alignment is reviewable rather than accidental. DNS changes are asynchronous, so readiness should be persisted with a last-checked timestamp rather than fetched during every signup.

Exercise the whole feedback path in staging with synthetic recipients supported by the selected transport. Confirm that an authenticated permanent-bounce event creates one suppression record even when delivered twice. Confirm that a transient event does not create a permanent suppression. Confirm that a suppressed signup produces no queued message. These three tests catch more integration mistakes than a screenshot of a received email.

Deployment order matters: create the suppression store and event consumer first, verify event authentication, then enable outbound traffic. Otherwise the system can emit messages before it can learn from failures. Keep a manual suppression path for abuse and support work, with an audit trail and a narrowly controlled removal operation.

For SMS onboarding, use a separate consent and suppression policy. Email authentication records do not establish permission to text, and CTIA messaging guidance treats consumer consent and opt-out handling as messaging concerns. Sharing an `isContactable` boolean across email and SMS erases that distinction.

## Does this add too much integration work?

The smallest credible implementation needs a domain-readiness lookup, a suppression lookup, an outbound queue, and one authenticated event consumer. That is more work than one `send()` call. It is also the minimum shape that closes the loop.

Keep the application-facing contract narrow and push transport-specific payloads into adapters. Evaluate an integration by how clearly it exposes authentication evidence, stable event identifiers, bounce classification inputs, retry behavior, and test fixtures. Count operational edges, not SDK lines. A short SDK example can still hide DNS setup, event verification, reconciliation, and replay handling.

A second objection is latency. Two indexed lookups before enqueueing are easier to reason about than remote validation in the signup request, and they can run within the worker that owns delivery. If throughput later demands caching, cache domain readiness briefly; keep suppression authoritative or define invalidation. The safety bias is asymmetric: stale domain readiness delays a welcome, but a stale suppression cache can contact a recipient the system already knows it should avoid.

The decision rule is crisp: choose an architecture whose failure events can become durable recipient state with the fewest translation layers. Domain authentication gets a tenant onto the track. The feedback loop keeps it there.

## Further reading and References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 3463, Enhanced Mail System Status Codes: https://datatracker.ietf.org/doc/html/rfc3463
- RFC 3464, Delivery Status Notifications: https://datatracker.ietf.org/doc/html/rfc3464
- CTIA Messaging Principles and Best Practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
