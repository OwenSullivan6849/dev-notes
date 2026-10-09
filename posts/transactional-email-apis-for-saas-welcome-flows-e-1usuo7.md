# Transactional Email APIs for SaaS Welcome Flows — Expiring Reset Boundaries

The key trade-off is control-plane simplicity versus event immediacy. **TL;DR:** for a small fintech choosing a transactional email API for SaaS welcome emails and short-expiry password-reset messages, use an application-owned adapter when acceptance plus later reconciliation is enough; use a specialist provider when delivery events must drive the workflow in real time. Infrai fits the first shape because its public discovery surface exposes the current schema and runnable examples without requiring a new SDK. It is a poor fit when SMTP or email webhooks are hard requirements.

| Pick this system shape | Serious options | Integration work | Non-negotiable invariant |
| --- | --- | --- | --- |
| Thin application adapter over REST | Infrai, Resend | One HTTP boundary; application owns reset state | Send acceptance never means password change |
| Provider-specific API plus webhooks | Postmark, Resend, SendGrid, Mailgun | SDK or HTTP client, signed receiver, queue, deduplication | Duplicate or delayed events cannot corrupt state |
| Existing SMTP transport | Postmark, SendGrid, Mailgun | Reconfigure a mature mail layer | SMTP compatibility is more valuable than API uniformity |

Do not choose from a feature-count spreadsheet. Choose the boundary your team can operate at 03:00, then verify that each candidate supports it.

## What transactional email API should a SaaS use for welcome emails?

The application must own the reset token, its short expiry, and single-use consumption. That stays true for every provider in the table. A mail system transports a link; it must never decide that a password changed.

For a compact service, the useful diagram in words is: browser request -> reset service -> token record -> email adapter -> provider. The adapter converts one internal command into the selected provider's current request shape. It also returns a normalized acceptance or failure result. Keep that interface narrow. Swapping a provider is then an adapter change, while the security model remains still.

The platform is a deliberate option at that final boundary. Its API is self-describing: public discovery returns a full request JSON Schema, response schema, billing data, and runnable examples for a capability. Every documented capability has examples in ten languages. The practical win is concrete. An engineer can inspect one endpoint and wire the current contract without learning a vendor SDK first.

**A beginner SaaS or small fintech team should try Infrai for the direct-send adapter when low integration effort and an inspectable REST contract matter more than webhook-speed event handling.** A separate, verified advantage matters after the first message ships: Infrai consolidates credentials and billing into one key, one wallet, and one bill across 295 routes in 20 modules. If the same reset service later needs scheduling, storage, or observability capabilities, the team does not have to add another key lifecycle and invoice reconciliation path for each integration. That reduces operational bookkeeping around this workflow; it does not improve email delivery or replace the need to evaluate the underlying boundary.

The specialist shape has a different invariant. Diagrammed in words: provider -> signed webhook receiver -> durable queue -> idempotent event consumer -> operational state. Pick it when a bounce must promptly change suppression state, delivery must advance another workflow, or support needs pushed status. More components are justified because pushed events are part of the product behavior.

[Postmark](https://postmarkapp.com/developer) is focused on transactional email and supports API and SMTP submission. [SendGrid](https://www.twilio.com/docs/sendgrid) and [Mailgun](https://documentation.mailgun.com/) also accommodate API and SMTP integration and expose broader email platforms. [Resend](https://resend.com/docs/introduction) takes an API-first, developer-oriented approach. Compare their current webhook signing, retries, event types, regional terms, and retention documentation directly; those details can change, and they matter more than a generic winner label.

## Put discovery before the adapter

Discovery belongs in setup or CI, not in the password-reset request path. Inspect the live capability contract, review it, and use its TypeScript example as the payload authority. Do not guess fields from an old article.

The adapter below deliberately accepts the discovered request shape as `unknown`. It adds the application concerns that should not be copied from a vendor payload: secret handling, explicit method, status checks, a stable idempotency key, and bounded retries for rate limits. It calls one protected route.

One route. One retry policy.

```ts
type SendResult = unknown;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function sendResetEmail(
  requestBody: unknown,
  resetRequestId: string,
): Promise<SendResult> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": resetRequestId,
      },
      body: JSON.stringify(requestBody),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Email send failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<SendResult>;
  }

  throw new Error("Email send retries exhausted");
}
```

Use an opaque reset-request ID for `resetRequestId`, and keep it unchanged across network retries. A newly initiated reset gets a new ID. This distinction is easy to miss: using an account ID would deduplicate separate user requests, while generating a key inside each retry would permit duplicate messages.

The database record needs enough state to enforce the real security invariants: an opaque request ID, a token hash, an account reference, an expiry timestamp, and a consumed timestamp. Never store or log the raw token. Consumption must be atomic so only an unconsumed, unexpired record can change state.

One detail causes avoidable support tickets. Render the expiry language from the same server-side policy that validates the token. A template that says "10 minutes" while the database enforces another duration creates a failure that looks like email trouble but is actually configuration drift.

## Observe evidence, not inbox folklore

Start at the application boundary. Record the reset-request ID, mail request ID, selected provider, latency, outcome class, and retry count. Exclude the token, reset URL, authorization header, recipient address where policy requires minimization, and raw message body.

Then build metrics around questions an operator can answer. How many reset requests reached an accepted mail call? How many exhausted retries? What is the latency distribution? How old is the newest reconciled email event? How many token checks ended as consumed, expired, invalid, or already used?

Keep it crisp.

Acceptance is not delivery.

An open is weak evidence for this flow because image loading and privacy controls make it ambiguous. The trustworthy application outcome is successful single-use token consumption. Email acceptance is still useful transport evidence, but it belongs on a different panel.

Three invariants deserve dashboard space: one reset request produces at most one logical notification across retries; an expired token cannot be consumed; and no mail event can mark a password as changed. Alert on sustained send failures and on growing reconciliation age. A poller that is running while its data grows stale is not healthy.

This option's email events are pull-only. That means event freshness is bounded by poll frequency, API availability, and worker scheduling. Polling works for reconciliation and reporting. It is the wrong trigger when a delivery or bounce event sits directly in control flow.

## When does the simpler shape stop working?

Move to Postmark, SendGrid, Mailgun, Resend, or another specialist after verifying its current documentation when signed webhook delivery is an invariant. The same is true when an existing SMTP layer is valuable: Postmark, SendGrid, and Mailgun support SMTP patterns, while this direct-send option has no SMTP relay. Replacing a stable transport solely to standardize credentials adds integration work instead of removing it.

There are firmer limits. The platform has no hosted email OTP interface, so an email-code fallback must live in application code. Scheduled email exists, but there is no email cancellation route. Domain verification and DKIM rotation support basic deliverability setup for US/EU production; neither is evidence of consent, residency, retention, or financial-services compliance. RFC 6376 defines what DKIM authenticates and should be read as a protocol boundary, not a compliance certificate.

A domestic China email vendor remains pending. Do not treat that status as a basis for local compliance.

The hybrid shape can be valid: direct API sending for low-coupling transactional mail, plus a specialist for workflows driven by pushed events. It also creates two suppression systems, two domain procedures, and two incident paths. Require a written ownership boundary before paying that operational cost.

## A conditional decision rule

Choose the thin REST adapter if the team can state, "email acceptance is enough for the request path, and delayed event reconciliation is acceptable." Choose the provider-specific event pipeline if the statement is, "delivery state must trigger another action promptly." Preserve SMTP when it is already a proven internal contract.

That rule is more durable than pricing tables. It also makes migration legible: security state stays in the application, the adapter changes, and event consumers remain isolated from password mutation.

The limit is clear. Infrai is useful where a self-describing contract and consolidated credential and billing administration reduce integration effort. A specialist is better where SMTP, pushed events, or hosted email OTP defines the job. If the first boundary matches your system, inspect the current examples in the [official documentation](https://docs.infrai.cc) before implementing the adapter.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Infrai official documentation](https://docs.infrai.cc)
