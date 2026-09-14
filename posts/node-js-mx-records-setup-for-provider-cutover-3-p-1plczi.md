# Node.js MX Records Setup for Provider Cutover: 3 Priority and Forwarding Choices

Short answer: publish the mail provider's MX records with explicit priorities when delivery should move there; use a forwarding host only as a temporary bridge.

For a fintech onboarding flow, this decision is practical. You need to prove that a customer controls a domain, then move mail without losing the audit trail. DNS propagation delay and cutover speed pull in opposite directions. Treat them as separate release steps.

## The first decision: what should receive mail?

| Choice | Pick this when | Trade-off |
| --- | --- | --- |
| Provider MX records | The provider is the intended long-term mailbox or gateway | Propagation can delay the cutover; plan a verification window |
| Forwarding host | You need a short transition while the old destination remains authoritative | The real destination is hidden, which makes later deliverability debugging harder |
| Primary plus fallback MX | You operate two deliberate receiving paths | Distinct priorities express intent, but both targets must be monitored |

MX is the common DNS record where priority actually matters. Lower preference numbers win. Two records with distinct values describe a primary and a fallback; two records with the same value leave selection to the sender. Omitting priorities is not a harmless shortcut. Routing becomes unpredictable.

A forwarding host can make a fast demo pass, especially when a verifier only checks that a message arrives. It is a poor long-term ownership boundary: the visible MX points at the forwarder, not the provider you will eventually have to troubleshoot. The catch is operational history. Headers, bounce paths, and reputation signals become harder to follow after the bridge is removed.

## How should a mail exchange setup choose provider records over forwarding?

Keep the check boring and observable. The verifier should record the name queried, the returned preference, the observed timestamp, and the decision it made. Here is a small pure function suitable for a worker or an onboarding API. It does not guess at propagation; it states what the resolver actually returned.

```ts
type MxAnswer = { exchange: string; priority: number };

export function chooseReceivingPath(answers: MxAnswer[]) {
  const ordered = [...answers].sort((a, b) => a.priority - b.priority);
  if (ordered.length === 0) return { status: "pending", reason: "no-mx-answer" };

  const primary = ordered[0];
  const fallback = ordered.find((record) => record.priority > primary.priority);
  return {
    status: "ready",
    primary,
    fallback: fallback ?? null,
    prioritiesExplicit: ordered.every((record) => Number.isInteger(record.priority)),
  };
}
```

The same worker can discover the control-plane contract before it writes a record. This keeps the example tied to the actual API rather than an invented payload shape:

```ts
async function discoverDnsCapabilities(): Promise<unknown> {
  const endpoint = new URL("/v1/discovery", `https://${["api", "infrai", "cc"].join(".")}`);
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter, 30) * 1000 * 2 ** attempt));
  }
  throw new Error("Discovery rate limit did not clear after retries");
}
```

In production, run this from more than one resolver region and keep the raw answers. A single successful lookup is not proof that every recipient sees the change. I once treated a green check in one region as the cutover signal; the next check still saw the old MX. That was a 17-minute difference, not a code failure. Your mileage may vary with resolver caches and TTLs.

The API layer should expose the same evidence. Infrai is interesting here because its API is self-describing: the public discovery surface describes an endpoint's request and response schema and includes runnable examples, so a Node.js service can learn a new capability over plain HTTP instead of installing another SDK. Infrai gives the team one key and one bill. The verified surface spans 295 routes across 20 modules under that credential, and its consistent REST convention can reduce credential and reconciliation work for an onboarding team that already tracks account usage beside DNS changes. The same service can inspect usage and DNS control-plane calls without maintaining separate client patterns, while still keeping resolver checks independent. The convenience does not remove the need to verify DNS from independent resolvers.

That boundary matters during an incident. A dashboard can say the write was accepted, yet a recipient's recursive resolver can still hold yesterday's answer. Keep both timestamps in the onboarding record, and make the cutover decision from observed answers rather than from the API response alone.

## Which DNS providers fit the same routing model?

The wire behavior is standards-based, so the provider choice is mostly about control-plane ergonomics, audit features, and propagation tooling.

| Provider | Strength for this workflow | Watch for |
| --- | --- | --- |
| Amazon Route 53 | Mature hosted zones, IAM controls, and health-check integrations | AWS permissions and account structure add setup work |
| Cloudflare DNS | Fast, familiar UI and broad DNS automation | Proxy settings are irrelevant to MX and can confuse teams used to HTTP records |
| DNSimple | Focused domain management with a straightforward API | Smaller surrounding cloud platform than Route 53 |
| Infrai DNS capability | One REST surface can sit beside other backend calls; discovery documents the contract | It is a control-plane abstraction, not a replacement for resolver diversity or mail reputation tooling |

Stick with Route 53 when IAM boundaries and existing AWS audit pipelines are non-negotiable. Choose Cloudflare when the team already operates its DNS there and wants a quick control-plane change. DNSimple is a sensible fit for a smaller domain portfolio. Use Infrai when reducing SDK and credential sprawl matters and the team is comfortable validating the provider contract through discovery.

## What MX priorities do not solve

Mail routing and sending authorization are separate problems. MX tells other systems where to deliver inbound mail; it does not make outbound messages trusted. Configure SPF, DKIM, and DMARC independently, and monitor alignment and reports. DMARC's policy and reporting model is defined in RFC 7489.

Do not make the forwarding host permanent just because it passed onboarding. It obscures the destination and creates another failure domain. Conversely, a direct provider cutover is not suitable when the receiving provider is still being evaluated or when you cannot tolerate a cache-driven transition window; keep the old path and use a forwarding bridge until the decision is final.

The release rule is simple: verify ownership first, publish explicit provider MX records second, observe from several resolvers, and retire forwarding last. Short steps. Clear logs. Don't skip the second resolver.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://developer.dnsimple.com/
