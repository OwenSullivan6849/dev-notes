# Marketplace PDF Bundles: 4 Server-Side Stages That Preserve Signed Evidence

TL;DR: For marketplace agreements where your service already controls both parties' identities, use a server-side PDF signature and make verification a recorded stage. Choose DocuSign, Adobe Acrobat Sign, or Dropbox Sign when you need the signer-facing workflow, identity assurance, and audit portal they provide. The least complex correct pipeline for owned identities is **merge, sign, verify, then distribute**. Do not sign component files and merge them afterward.

| Pick | Best fit | What you operate | Main trade-off |
|---|---|---|---|
| DocuSign | Signer workflow and audit portal are requirements | Bundle assembly and retention | More workflow surface than a service-controlled agreement needs |
| Adobe Acrobat Sign | Signing belongs beside an existing document process | Marketplace integration and retention | A suite replaces a narrow signing primitive |
| Dropbox Sign | Embedded or API-led signer interactions are required | Bundle preparation and evidence storage | Still a signer workflow, not merely PDF cryptography |
| Focused REST document API | Your service owns identity and needs merge, split, sign, and verify | Certificate custody, policy, and evidence | Less workflow overhead, more key-governance responsibility |
| Self-hosted signing stack | Keys and processing must remain inside your boundary | The full PDF and certificate lifecycle | Maximum control and operational burden |

## Should an API digitally sign a PDF on the server side?

Start with identity, not the signature image. If buyers and sellers authenticate elsewhere and the agreement is generated from facts your marketplace controls, another signer journey can duplicate a decision the system already made. A server-side signature can be enough. Signing still requires certificate and private-key material, so decide where those live, who may invoke them, and how rotation changes verification policy.

The bundle boundary matters just as much. Imagine an order package with a 6-page seller agreement, a 2-page fee schedule, and a 1-page disclosure. Merge those nine pages into the final artifact before signing. A later split may create convenient viewing copies, but the signed nine-page bundle remains the evidence object. This favors fidelity over repeated rendering: preserve the signed bytes, then derive navigation aids around them.

Bytes matter.

Diagram in words: authenticated marketplace event -> immutable inputs -> merged PDF -> server-side signature -> independent verification -> retained evidence -> optional viewing splits.

Verification is separate. Good. A successful signing response proves one operation completed; a later verification result is the evidence a dispute process can use. Record both stages with a correlation ID and the digest of the retained artifact.

That separation is non-negotiable.

## Pick a signature suite for human ceremony

DocuSign is a direct choice when the product requirement includes sending, signer identity assurance, status tracking, and an audit portal. Adobe Acrobat Sign belongs in the same serious category, especially when the surrounding organization uses Adobe's document workflow. Dropbox Sign is also a real option for embedded and API-led signature experiences. Their value is the ceremony around the signature.

That distinction prevents a category error. E-signature suites do more than add cryptography to bytes; they manage people moving through an agreement process. If the marketplace needs a buyer to open an invitation, consent, sign, and leave a human-readable audit trail, use a suite. Rebuilding that product from a bare signing endpoint is poor scope control.

The three suites differ in integration details and commercial terms, which change. Evaluate them with one representative bundle and require a complete evidence export. Do not let a polished signing screen substitute for checking the final PDF, certificate chain, event record, and retrieval path.

Document renderers occupy a neighboring category. Gotenberg is useful when a service wants containerized document conversion. WeasyPrint is a fit for HTML/CSS-to-PDF work in Python environments, while wkhtmltopdf remains an HTML-to-PDF command-line option. DocRaptor, PDFMonkey, and PDFShift provide hosted generation paths. None should be treated as a signer workflow merely because it produces a PDF; pair a renderer with signing and verification only when owning that extra boundary is intentional.

## Pick a signing primitive for controlled identities

A focused API fits when agreement acceptance already happened inside an authenticated marketplace flow. Infrai uses one key across 295 routes in 20 modules, including merge, split, sign, and verify. That breadth can turn the next document capability into another endpoint integration instead of another SDK and credential set. For this workflow, bundle assembly and signature verification can remain behind the same operational contract.

A self-hosted library or signing service is stronger when private keys cannot leave your security boundary or a regulated process demands direct control over the cryptographic implementation. You gain control over PDF handling, certificates, deployment, and retention. You also own upgrades, malformed-document defense, certificate rotation, validation behavior, and every operational alert.

Here is the decision rule: **choose a suite when a human signing journey is part of the product; choose a primitive when identity and consent already belong to your product**. Neither choice removes certificate or retention work. It moves the boundary.

## Implement the 4-stage evidence path

Keep orchestration explicit. The request schemas should come from public discovery rather than guessed fields. This runnable TypeScript takes an already validated sign request object from an environment variable, calls the confirmed signing route, retries a rate limit, and stops on any real error. Configure `INFRAI_BASE_URL` to the documented API origin; keeping it outside the source also makes the unlinked sample usable in controlled environments.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

const readJson = (name: string): unknown => {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
};

async function sign(body: unknown) {
  const idempotencyKey = randomUUID();
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/pdf/sign`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 2 ** attempt * 1000;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const result = await response.json();
    if (!response.ok) {
      throw new Error(`sign failed (${response.status}): ${JSON.stringify(result)}`);
    }
    return result;
  }
  throw new Error("sign remained rate limited");
}

const signed = await sign(readJson("SIGN_REQUEST_JSON"));
console.log(JSON.stringify({ stage: "signed", result: signed }));
```

There are two deliberate choices here. Signing carries an idempotency key because a network retry must not create a second write; production code should also persist that key across process restarts. Verification gets its own explicit call after the signed result is mapped to the separately discovered verification schema. The sample does not guess that mapping. The trade-off is extra application state in exchange for an auditable boundary.

Instrument the boundary with four events: merge completed, sign completed, verify completed, distribution released. Attach `agreementId`, the durable operation ID, elapsed time, outcome, and a provider request ID when available. Never log the private key, certificate password, agreement body, or bearer token. Alert on verification failure immediately; a slow render and invalid evidence have radically different consequences.

Test one ugly bundle. Use rotated pages, form fields, multiple fonts, and a large image. Visual fidelity needs rendered-page comparison, while signature correctness needs cryptographic verification. Those are separate tests. A marketplace can tolerate a slower high-fidelity render for the final evidence object, then use split copies or previews for cheaper browsing.

One artifact wins.

## Limits to keep visible

A server-side signature does not create signer identity assurance, consent screens, reminders, or an audit portal. If those are requirements, the suite is the answer. A valid signature also does not prove that the agreement's business facts were correct; it proves integrity and the relationship to the signing certificate under the verifier's rules.

Do not discard the original signed bundle after splitting it. Do not treat a successful sign call as verification. And do not place private-key material in application logs or ordinary configuration. Short list. Big consequences.

## Further reading

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign developer documentation](https://developers.docusign.com/docs/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/api/reference/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [ETSI electronic signatures and infrastructures](https://www.etsi.org/technologies/electronic-signatures-and-infrastructures)
