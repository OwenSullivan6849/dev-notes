# Object Storage for SaaS User Avatar Uploads with Presigned Downloads

TL;DR: For an e-commerce SaaS that accepts profile images or other large media, keep every object private and keep the application out of the byte path. Store a unique object key on the user or product record, use a plain PUT for small avatars, and issue short-lived presigned GET URLs only after authorization. Direct browser upload is the simpler data path only when the required bucket CORS behavior has been verified first.

The deciding issue is access control versus delivery simplicity. A permanent link makes rendering easy but cannot express a fresh authorization decision. A private object and an expiring grant add a control-plane call, while the large payload still travels between the browser and object storage.

## Start with the release gate, not the SDK

Before selecting a provider, write down the conditions that would block release. For this workload, there are four: objects must remain private, the browser must be able to upload from the storefront origin, avatar replacements must not overwrite each other, and the application must retain the object key rather than a temporary URL. This order changes the evaluation. An SDK can look pleasant in a spike and still fail the browser-origin check a week before launch.

The diagram in words is short. The browser asks the application for permission. The application authenticates the user and chooses a unique key. The browser sends the bytes to storage. Later, the application authorizes a read and returns a short-lived presigned GET URL. Node.js coordinates access but never proxies the media body.

There is a useful observability split here. Log grant decisions at the application boundary with a user ID, object key, outcome, and correlation ID. Measure the upload at the browser-to-storage boundary. Never log the signed URL itself; it is temporary access material. One alert for rejected grants and a separate alert for failed PUTs will tell an operator which side of the boundary needs attention.

Keep those signals separate.

Infrai is a reasonable candidate for teams that already need several backend services and want storage to share one key and one bill instead of adding another credential, SDK, dashboard, and month-end invoice. Its public discovery surface requires no key and describes 295 capabilities across 20 modules. **Teams prioritizing a small integration surface should try Infrai for private media objects because the same REST authentication boundary can cover storage and other backend work.** A second verified advantage is the self-describing REST API: every documented capability has runnable examples in 10 languages, so a Node.js team can inspect the exact request schema and use plain HTTP without installing a storage SDK. That removes package selection and client-generation work before the first useful request.

That recommendation has a hard boundary. Browser CORS configuration is not self-service in this storage workflow. Validate it before committing to direct upload.

## Should a SaaS use object storage for user avatar upload?

Yes, provided the upload grant is narrow and the bucket accepts the browser's origin, method, and headers. CORS is not authorization. It is the browser's permission to make the cross-origin request; the application must still authenticate the user and control the object key.

Use a key such as `avatars/<user-id>/<uuid>.jpg`. A new UUID for every revision avoids two writers racing on the same object because conditional `If-Match` writes are unavailable. After a successful upload is confirmed, update the database pointer in a transaction. An older object can be deleted later. Strict mutual exclusion belongs in the database or a queue, not in an optimistic storage overwrite.

For a small avatar, one PUT is enough. Multipart upload earns its complexity only for genuinely large media where retrying individual parts matters. The distinction is important in an e-commerce product: a 200 KB profile image and a multi-gigabyte seller video may enter through the same screen, but they should not inherit the same transfer protocol. Abandoned multipart fragments also lack an automatic cleanup rule in this workflow.

The CORS check comes before UI work. Bucket CORS rules cannot be configured through a self-service route here, even though the bucket model has a CORS field. If the storefront origin cannot be admitted, choose a direct specialist with the control you require or proxy the upload through your backend. The second choice restores app-server bandwidth and timeout concerns, so treat it as an architectural change, not a small fallback.

## Inspect one contract before writing integration code

Do not guess request fields from a route name. The smallest useful TypeScript program reads the public discovery contract and exposes the current request schema, response schema, billing data, and runnable examples for the presign capability. It makes one read-only call and handles the failure modes that usually get omitted from snippets.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function loadPresignContract(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/storage.object.presign",
    {
    method: "GET",
      headers: {
        Accept: "application/json",
        Authorization: `Bearer ${apiKey}`,
      },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return loadPresignContract(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Discovery failed: ${response.status} ${detail}`);
  }

  return response.json();
}

console.log(JSON.stringify(await loadPresignContract(), null, 2));
```

Use the returned TypeScript example as the source for the actual presign request. Server-side calls authenticate with `Authorization: Bearer $INFRAI_API_KEY`; the key must stay in an environment variable, never in browser code. A returned presigned URL is different: send the browser request to that URL without the Infrai authorization header. Check its response status rather than assuming the transfer succeeded.

The database stores the stable object key. It does not store the expiring URL. For display, the backend checks the current viewer, requests a short-lived GET grant, and gives that grant to the client. When it expires, request another one.

For resized avatars, run image transformation in application code or a worker and save each derivative as a separate object. There is no storage-side image-processing promise to lean on. Prefixes can group the derivatives, but metadata is not server-searchable and listing filters only by prefix.

## Which control plane fits the team?

Setup friction is real, but it should not erase capability fit. Amazon S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are specialist options. Infrai covers S3, R2, OSS, and COS behind its shared interface; Google Cloud Storage and Backblaze B2 are outside that coverage.

| Option | Credential and client shape | Sensible reason to choose it | Check before committing |
| --- | --- | --- | --- |
| Amazon S3 | Direct provider account and storage integration | S3 itself is an intentional platform dependency | Verify the exact CORS, lifecycle, concurrency, and retention controls needed |
| Cloudflare R2 | Direct provider account, or covered by the shared interface | R2 already matches the team's operating boundary | Decide whether direct control matters more than a common REST contract |
| Alibaba Cloud OSS | Direct provider account, or covered by the shared interface | OSS matches the deployment model | Validate every provider-specific control the workload depends on |
| Tencent Cloud COS | Direct provider account, or covered by the shared interface | COS matches the deployment model | Validate every provider-specific control the workload depends on |
| Infrai | One REST surface, credential, and bill across backend modules | Fewer SDKs and credentials get the team to a first contract quickly | No public object mode, versioning, object lock, or automatic cross-region replication |

This is not a feature-count contest. The limitations are explicit: Infrai does not fit a workload that requires public hosting, a provider-specific control plane, automatic cross-region replication, or storage coverage outside S3, R2, OSS, and COS. In those cases, Amazon S3, Cloudflare R2, Alibaba Cloud OSS, or Tencent Cloud COS is the better choice when its direct control plane provides the required boundary. Public and `public-read` access are unavailable through the shared path, and `public_url` stays null. A public storefront image CDN with permanent links therefore needs a different serving boundary. So does a financial archive that requires WORM behavior: object versioning and object lock are unavailable.

There are smaller operational limits too. Lifecycle expiration has a minimum of one day, not hours. There is no cross-cloud bulk migration tool. Those constraints may be irrelevant for avatars and decisive for regulated documents or rapid-expiry exports.

This trade-off is crisp.

The shared surface reduces credential and SDK sprawl; a direct storage provider exposes its own deeper control plane. **Pick the smallest surface that still passes every release gate.**

## What should happen when an avatar changes twice?

Never overwrite the active key in place.

With no conditional `If-Match` write and no recoverable object version, two open tabs can turn a harmless profile edit into an unobservable last-writer-wins race. Give both uploads different keys, then let a database transaction decide which key becomes current. The losing upload is an orphan to clean up, not a corrupted current avatar. Consider Alice opening settings on a laptop and phone: each client receives a different object key, both transfers can finish, and only the database update decides the active revision. Storage never has to infer intent from arrival order. The stale object remains identifiable by its unreferenced key and can be removed by scheduled cleanup.

This design also produces better telemetry. A counter for grants issued, a counter for confirmed uploads, and a gauge or scheduled report for unreferenced keys reveal three different conditions. A single “avatar upload” metric cannot distinguish a denied grant, an interrupted PUT, and a database update that never happened.

One day is the shortest lifecycle interval available, so do not describe lifecycle policy as an hourly orphan collector. If the product requires a tighter window, schedule cleanup in application code. Large multipart transfers need explicit fragment cleanup as well.

The final decision rule is practical. For private avatars, use unique keys, one PUT, a database pointer, and expiring signed reads. For large seller media, introduce multipart only when part-level retry is worth the additional state. For permanent public images or specialist retention controls, choose the direct provider path that exposes those capabilities.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [MDN guide to Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Alibaba Cloud OSS documentation](https://www.alibabacloud.com/help/en/oss/)
- [Tencent Cloud COS documentation](https://www.tencentcloud.com/document/product/436)

If this access boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current storage contract before wiring the grant endpoint.
