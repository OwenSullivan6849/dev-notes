# Erasing Customer Vectors — Auditable Deletion for Grounded Game Support

Deleting a player's account should trigger a tenant-scoped erasure workflow, not a blind similarity-search cleanup. The deciding constraint is proof: every vector must be traceable to an owned source record, and the system must show that neither retrieval nor citations can surface that player's material after deletion.

**TL;DR:** assign an immutable customer ID and source ID before chunking, copy both into every vector's metadata, record each successful write in an ingestion manifest, and erase by customer ID through an idempotent job. Then test the read path with the deleted customer's exact source IDs. A successful delete response alone is weak evidence.

## How should GDPR account deletion remove all customer vectors?

A game support document rarely stays one record. A player's crash report, save-state note, or moderation appeal may become 10 chunks, and a later re-index can create another generation with different vector IDs. If deletion code knows only the original document ID, it can miss the derived copies. The before model is deceptively tidy: account to document to vector. The useful after model is a lineage graph: account to sources, sources to ingestion generations, and each generation to chunks. Citations travel back across the same graph. Erasure should, too. This changes the unit of control. Chunk IDs are storage details. `customerId` is the deletion boundary, while `sourceId` preserves grounding and lets an answer cite the game-support artifact from which a chunk came. Keep those values as exact-match metadata; don't rely on their presence in embedded text. Similarity is probabilistic, ownership isn't, and a delete operation needs the latter.

That's the trap.

## Build one lineage-aware deletion path

The adapter below stays deliberately generic. Its three operations express the contract the application needs: delete all vectors matching an exact ownership filter, list the manifest entries created during ingestion, and mark the erasure job complete only after verification. The manifest belongs in a transactional system of record, separate from the vector index, because it is the inventory used to audit derived data.

```ts
type VectorFilter = { customerId: string };

type ManifestRow = {
  customerId: string;
  sourceId: string;
  generationId: string;
  vectorIds: string[];
};

interface VectorIndex {
  deleteByFilter(filter: VectorFilter): Promise<void>;
  fetch(ids: string[]): Promise<Array<{ id: string }>>;
}

interface ManifestStore {
  listByCustomer(customerId: string): Promise<ManifestRow[]>;
  markErased(customerId: string, erasedAt: string): Promise<void>;
}

export async function eraseCustomerVectors(
  customerId: string,
  index: VectorIndex,
  manifest: ManifestStore,
): Promise<{ checked: number }> {
  if (customerId.trim().length === 0) {
    throw new Error("customerId is required");
  }

  const rows = await manifest.listByCustomer(customerId);
  const expectedIds = [...new Set(rows.flatMap((row) => row.vectorIds))];

  // Exact ownership metadata is the primary delete boundary.
  await index.deleteByFilter({ customerId });

  // The manifest turns an acknowledgment into a testable postcondition.
  const survivors = await index.fetch(expectedIds);
  if (survivors.length > 0) {
    throw new Error(`Erasure incomplete: ${survivors.length} vectors remain`);
  }

  await manifest.markErased(customerId, new Date().toISOString());
  return { checked: expectedIds.length };
}
```

At ingestion time, write metadata shaped like `{ customerId, sourceId, generationId }` on every chunk and append the returned vector IDs to the manifest. Do not mark a manifest row committed before the vector write succeeds. If the process dies between those writes, reconciliation must compare the pending generation with the index and either finish it or erase it. That awkward boundary is where orphan vectors are born. Make the erasure worker idempotent. A retry may find no vectors, and that should still be success once verification finds no known IDs. Serialize ingestion and erasure for the same customer, or use an account lifecycle state that rejects new ingestion after deletion begins. Otherwise a worker can add a fresh chunk milliseconds after the deletion filter runs. The trade-off is a little more coordination on the write path in exchange for a deletion boundary that can't race an active ingest.

Retries are normal.

## Grounding makes the verification stricter

RAG combines retrieval with generation, so clearing the vector index is necessary but not the entire read-path test. Cached retrieval results, answer caches, citation stores, and exported evaluation fixtures can retain derived or copied material. Inventory those stores by the same customer boundary and give each one an erasure handler.

The acceptance test should query with a distinctive phrase from each deleted source and inspect both retrieved chunks and citations. It must return no result owned by the deleted customer. Also issue a normal query against shared game documentation. That second assertion catches the opposite failure: an over-broad filter that deletes global patch notes or help articles alongside private player material.

Log counts, not content. Useful fields include a pseudonymous erasure job ID, deletion phase, manifest row count, expected vector count, survivor count, retry count, and completion timestamp. Never put the player's prompt, document text, or raw account identifier in the audit event. A counter for incomplete verification and an alert on jobs stuck between delete and verified completion make this operational instead of ceremonial.

**The key invariant is simple:** after the erasure job completes, no retrieval result or citation may cross from the deleted ownership boundary into an answer. Test that invariant at the API used by the game-support assistant, not only against an index administration endpoint.

Treat the metadata filter as the broad sweep and the manifest as the precise checklist. Either one alone is weaker. A manifest can miss a write if bookkeeping failed; a filtered delete can be misconfigured or scoped to the wrong partition. Using both creates two independent signals.

Schedule reconciliation before an erasure request ever arrives. Enumerate index metadata through the adapter, then compare it with committed manifest generations. Quarantine entries with no source lineage. Also check the reverse direction: a manifest vector ID that cannot be fetched may indicate an interrupted replacement or retention process. These are data-quality signals as much as privacy signals.

Do not patch a lineage gap by running semantic searches for the customer's name, email address, or memorable phrases. Those values may be absent, normalized, shared by another person, or split across chunks. Worse, this technique turns a deterministic ownership operation into a fuzzy guess. Fix lineage, then repeat deletion.

Now test it.

## Should backups be rewritten immediately?

Usually, the practical design is to prevent erased records from returning during restoration rather than editing every backup in place. The exact policy depends on the organization's legal basis, retention obligations, and backup system, so legal and security owners must set it. Article 17 of the GDPR defines the right to erasure and its exceptions; engineering should encode the approved policy rather than invent one.

A restoration runbook needs an erasure ledger that can be replayed after restoring an older snapshot. Keep that ledger access-controlled and minimal: a stable pseudonymous subject key, the affected stores, and the deletion state can drive replay without preserving the content that was meant to disappear. Test the sequence. Restore, replay erasures, rebuild derived indexes, then run the same retrieval-and-citation checks before serving traffic.

There is a real trade-off here. Longer-lived lineage and erasure records improve auditability, but those records are themselves personal-data-adjacent and need a defined retention policy. Store only what proves and executes deletion. More detail is not automatically better.

## Definition of done

An HTTP success code is not the finish line. Completion means ingestion is closed for the account, all registered stores have run their handlers, known vector IDs cannot be fetched, ownership-filtered retrieval returns nothing, distinctive-source queries yield no private citation, shared support content still works, and the audit event contains no deleted content.

Keep the workflow small enough to rehearse. Trigger it in a staging index with a synthetic player, force a retry between deletion and verification, and exercise a restore from backup. One crisp dashboard should show jobs started, jobs verified, failures by phase, retries, and age of the oldest unfinished job.

Then the system has evidence.

## References

- https://arxiv.org/abs/2005.11401
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://opentelemetry.io/docs/specs/otel/logs/
