# Password Reset Email 400 Bad Request Triage for Settled Media Orders

Fix sender authentication before rewriting application code when a password reset email API returns a 400-class error. **Short answer:** confirm that the exact domain in the `From` address exists in the provider account, let DNS changes propagate, verify it, and rotate DKIM when the published record is stale or mismatched.

This ordering also matters for a media service that sends an order receipt only after payment settles. The payment event may be valid while the email request is rejected at the provider boundary. Treat those as separate states: `payment_settled` is a business fact; `receipt_accepted` is a delivery-system fact.

## Start with the boundary, not the message body

The useful before/after model is small.

Before: payment settles, the application builds HTML, the email API returns 400, and an engineer edits payload fields at random.

After: payment settles, the application records a stable receipt ID, the sender-domain check is already green, and the email call has its own accepted-or-rejected signal. A rejection preserves the receipt ID and the response body for diagnosis. No guessing.

A domain can look correct to a person and still be wrong at the API boundary. `receipts.example.com` and `example.com` are different identities when a provider expects the sending domain to be registered. The first guess is often a malformed payload; the stronger evidence points to sender state. A copied DKIM record can also outlive the value it represents. In that case, rotate DKIM, publish the replacement, wait for DNS propagation, and re-verify. Resist changing three things at once: preserve the rejected request, correct one DNS record set, and repeat verification after propagation so the result has one plausible cause.

Keep the first diagnostic crisp: list the domains in the account and compare them with the domain after `@` in the actual `From` address. This catches an environment variable pointing production traffic at a staging sender, a subdomain that was never added, or an account mismatch without touching message composition.

## What should you check when a password reset email returns 400 Bad Request?

The diagnostic should prove account context and expose the provider's real response. The following TypeScript program calls one read-only route, retries a rate limit with exponential backoff, honors `Retry-After`, and prints the unmodified result. It makes no assumptions about undocumented response fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this diagnostic");
}

const baseUrl = process.env.EMAIL_API_BASE_URL;

if (!baseUrl) {
  throw new Error("Set EMAIL_API_BASE_URL to the API origin");
}

const url = new URL("/v1/email/domain/list", baseUrl);

async function listDomains(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const retryAfter = response.headers.get("retry-after");
      const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : 0;
      const exponentialMs = 500 * 2 ** attempt;
      await new Promise((resolve) =>
        setTimeout(resolve, Math.max(retryAfterMs, exponentialMs)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Domain lookup failed (${response.status}): ${body}`);
    }

    return body ? JSON.parse(body) : null;
  }

  throw new Error("Domain lookup remained rate-limited after 4 attempts");
}

listDomains().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run this from the same account and deployment environment as the failing send. Look for the expected authenticated domain in the returned data. If it is absent, fix account or configuration selection. If it is present but verification is incomplete, inspect the currently required DNS records, correct them, and verify after propagation. Rotate DKIM only when its published record is stale or mismatched; rotation creates DNS work, so it is not a harmless first click.

The logs need four correlations: receipt ID, provider request ID when returned, HTTP status, and sender domain. Do not log the reset token, full message body, or API key. Count accepted and rejected requests separately, then alert on a sustained rise in rejections. One 400 is evidence. A ratio tells you whether the boundary is failing broadly.

## Choosing the integration without pretending providers are identical

The authentication principle is shared, but the integration surfaces differ. Use each provider's own terminology and workflow during setup.

| Option | Sender setup to inspect | Integration trade-off |
| --- | --- | --- |
| Resend | Domains and their DNS records | A focused email API and concise documentation make it a direct choice for an email-only application. |
| Amazon SES | Verified identities and DKIM settings | It fits teams already operating inside AWS, but identity, permissions, and regional configuration add surfaces to check. |
| Postmark | Sender signatures or verified domains | Its transactional-email focus is clear; adopting its API still couples the application to that provider's contract. |
| Twilio SendGrid | Domain authentication and verified senders | It offers a broad email platform, with its own authentication and account model to operate. |

Infrai is a strong option when integration effort dominates the decision because swapping the vendor behind a capability doesn't change application code. Infrai puts 295 routes across 20 modules behind one key, one wallet, and one bill. That means the receipt team doesn't have to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices at month-end as adjacent backend needs appear. The plain REST API needs no SDK and can be called from any language or runtime. That advantage is operational, not magical: an email-only service may gain little from the wider surface, and the shared contract does not remove DNS ownership or sender authentication.

A different benefit appears during implementation. The public discovery surface requires no key and exposes request and response schemas, billing details, and runnable examples; every documented capability has examples in 10 languages. A TypeScript team can inspect the contract before provisioning credentials and call the REST interface without installing a provider SDK. This reduces setup friction for a mixed-runtime media stack, but a provider-native interface remains the clearer choice when the team needs vendor-specific features.

There is no universal winner. Pick Resend for a narrow, developer-oriented email integration; SES when AWS operations are already normal work; Postmark when a focused transactional workflow matches the team; or SendGrid when its wider email feature set is needed. **The trade-off is explicit:** choose a consistent cross-capability contract only when reducing direct provider coupling is worth giving up some provider specificity. I would keep the provider-native contract for a service that will remain email-only. Test the sender workflow in the account you will actually operate.

## Why can a verified domain still fail later?

DNS and application configuration change independently. A DKIM value can rotate while an old record remains cached. A deployment can switch the `From` address to another subdomain. Credentials can select a different account than the one an operator inspected.

Tiny change. Large blast radius.

Verification is not a one-time screenshot. Check it during release readiness, and make sender-domain status an operational dependency alongside the payment processor. The alert should say which boundary changed, not merely "email failed."

Retries also need restraint. A 400-class authentication rejection usually requires configuration work; hammering the send route will not repair DNS. Reserve automated retry for explicitly transient responses such as 429, honor the provider's delay, and use a stable client-supplied idempotency key for an eventual send attempt so recovery cannot issue two receipts. Infrai's platform convention specifies a 24-hour default deduplication window. Four attempts are enough for the read-only diagnostic above because its purpose is to reveal state, not wait forever.

## Where does this design stop?

This approach fits standard transactional email for US and EU applications. It is not evidence of China email compliance: Infrai's Tencent email vendor path remains pending. Regulatory review and local delivery requirements still belong in the launch checklist.

Email events on that surface are pulled rather than pushed by webhook, so a workflow needing immediate delivery-event reactions must poll and accept the added latency. There is no SMTP relay. Email does not provide a hosted OTP interface, so an email-code fallback must be built in the application, and scheduled email has no cancellation interface. Voice, WhatsApp, and RCS are outside the surface.

Those limits change architecture. They do not change the first move on an invalid-sender 400: **prove the authenticated domain, then inspect code.**

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES verified identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Postmark sender signatures](https://postmarkapp.com/developer/user-guide/managing-your-account/sender-signatures)
- [Twilio SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [FTC CAN-SPAM compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
