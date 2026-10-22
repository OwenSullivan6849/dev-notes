# Scoped Room State API Explained (Multiplayer Game Reconnect Trade-offs)

The decisive trade-off is where recovery truth lives. Keep authoritative lobby and room state on the server, issue short-lived tokens scoped to one video room, and make reconnect a versioned resynchronization rather than a replay of every missed message. That choice costs a state store and a backfill path. It also gives a fintech team one place to reject stale membership, expired authorization, and out-of-order updates.

**TL;DR:** use a snapshot plus an ordered change stream when clients need quick recovery and an inspectable state transition. Use a snapshot-only API when room state is small. Use an event log when exact intermediate actions are the product requirement, not merely a recovery mechanism.

| Recovery model | Pick it when | Main cost | Reconnect behavior |
| --- | --- | --- | --- |
| Latest snapshot | The room is small and only current truth matters | Clients replace local state | Fetch one versioned document |
| Snapshot plus bounded changes | Clients benefit from recent deltas but can reset safely | Two response shapes and retention rules | Apply contiguous changes or fall back to a snapshot |
| Durable event log | Every accepted action must remain reconstructable | More storage, replay logic, and schema discipline | Resume from a durable position, then rebuild |

For a multiplayer game lobby that launches a scoped video session, the middle option is usually the useful starting point. This approach works cleanly behind a small Node.js API because the correctness contract stays in the store, not in one server process. It preserves a crisp reset path. It does not pretend that a transient socket is a database.

## How Should a Multiplayer Game Lobby API Recover Room State?

A snapshot-only contract is the smallest serious option. The server returns the lobby identifier, current phase, participants, video-room binding, and one monotonically increasing version. On reconnect, the client discards its cached room view and renders that snapshot. Pick this when the entire state is cheap to transfer and nobody needs to animate the missing transitions.

The catch is burst load. A network flap can make many clients request the same document together, so the handler and store need ordinary load protection. Instrument snapshot bytes, latency, and concurrent reconnects. Averages hide the moment that matters.

Snapshot plus bounded changes adds a cursor. A reconnecting client presents its last applied version. The server returns only contiguous changes when that cursor still falls inside the retained window; otherwise it returns a fresh snapshot. **The fallback is part of the protocol, not an error case.** This model fits a lobby where joins, leaves, ready-state changes, and the creation of a video-room binding are useful to apply incrementally, while the latest membership remains the final authority.

A durable event log is justified when the sequence itself carries business meaning. Think accepted commands that must be independently audited or reprocessed. It demands stable event schemas, explicit idempotency, and a defined rebuild procedure. Do not choose it merely because a socket can drop. That is a costly answer to a smaller question.

Here is the diagram in words: command enters the room service; authorization checks the caller's lobby membership; one transaction advances room state and its version; a publisher emits the committed change; connected clients receive it; reconnecting clients ask the recovery endpoint for changes after their last version. The video connection sits beside this path. Its token grants access to one bound room, while lobby state decides whether that grant may be minted.

## Reconnect Is a State Transition, Not a Socket Event

Treat `connected` as transport status. It does not prove that a client has current room state. A useful client state machine is `offline -> recovering -> live`, and only the recovery response may move it to `live`.

That distinction prevents a nasty ordering bug. Suppose the client last applied version 41. It reconnects and receives live version 45 before backfill versions 42 through 44. If it renders 45 and then blindly applies the older changes, local state moves backward. Instead, buffer live messages during recovery, apply a contiguous response beginning at 42, and then drain buffered messages in version order. Any gap triggers a snapshot reset.

Short rule: gaps reset.

Retries also need names. A command such as `set-ready` should carry a client-generated operation ID so the server can return the prior result when the same authenticated caller retries it. A recovery request is naturally repeatable because it reads after a version. A token-mint request needs stricter handling: validate current membership each time, scope the result to the bound room and permitted action, and never infer authorization from an old socket connection.

WebRTC defines the browser-side real-time connection interfaces and media behavior, but an application still owns its signaling and authorization design. Keep the peer-connection lifecycle separate from lobby recovery. A successful media connection cannot repair missing lobby versions, and a current lobby snapshot cannot prove that a media credential is still valid.

## A Minimal Versioned Implementation

The following TypeScript keeps the boundary intentionally small. The store is generic. A production adapter can back the same contract with a transactional database and a bounded change table.

```ts
type Member = {
  userId: string;
  ready: boolean;
};

type RoomState = {
  lobbyId: string;
  version: number;
  phase: "waiting" | "active" | "closed";
  videoRoomId?: string;
  members: Member[];
};

type RoomChange = {
  version: number;
  kind: "member-joined" | "member-left" | "ready-changed" | "video-bound";
  payload: Record<string, string | boolean>;
};

type Recovery =
  | { mode: "snapshot"; state: RoomState }
  | { mode: "changes"; fromVersion: number; toVersion: number; changes: RoomChange[] };

interface RoomStore {
  readState(lobbyId: string): Promise<RoomState | undefined>;
  readContiguousChanges(
    lobbyId: string,
    afterVersion: number,
    throughVersion: number,
  ): Promise<RoomChange[] | undefined>;
}

export async function recoverRoom(
  store: RoomStore,
  lobbyId: string,
  afterVersion?: number,
): Promise<Recovery> {
  const state = await store.readState(lobbyId);
  if (!state) throw new Error("ROOM_NOT_FOUND");

  if (afterVersion === undefined || afterVersion < 0 || afterVersion > state.version) {
    return { mode: "snapshot", state };
  }

  if (afterVersion === state.version) {
    return {
      mode: "changes",
      fromVersion: afterVersion,
      toVersion: state.version,
      changes: [],
    };
  }

  const changes = await store.readContiguousChanges(
    lobbyId,
    afterVersion,
    state.version,
  );

  if (!changes || changes[0]?.version !== afterVersion + 1) {
    return { mode: "snapshot", state };
  }

  return {
    mode: "changes",
    fromVersion: afterVersion,
    toVersion: state.version,
    changes,
  };
}
```

This function deliberately refuses to synthesize partial history. The store either supplies a contiguous range through the version observed in the snapshot read, or the client receives current truth. In a real transaction boundary, the state update and appended change must commit together. Publishing can happen afterward, keyed by the committed version, because recovery does not depend on successful delivery of that transient notification.

Token issuance belongs behind a separate authenticated command. Its input should identify the lobby, not accept an arbitrary media-room identifier from the browser. The service reads current membership, resolves the server-owned binding, and creates claims limited to that room and the action needed by the participant. Never place a reusable signing secret in lobby state or in client code.

## Observe the Gap, Not Just the Connection

Connection counts are comforting and incomplete. The operational question is whether clients converge on current state after interruption.

Record structured events for recovery mode, requested version, returned version, change count, and a machine-readable reason for snapshot fallback. Do not log the scoped token. A trace can link the authenticated recovery request to the state-store read and subsequent token-mint command without copying credentials into telemetry.

Four signals expose most mistakes:

- recovery latency, split by snapshot and changes;
- snapshot fallback count, split by cursor-expired, cursor-ahead, and noncontiguous-history reasons;
- version-gap count observed by clients or the delivery edge;
- token denial count, split by membership, room phase, and authorization scope.

Alert on sustained failure to converge, not on every reconnect. A reconnect is expected behavior on mobile and changing networks. Repeated gaps for the same lobby, a rising fallback ratio, or recovery that never reaches `live` indicates a contract or retention problem. Pair the alert with the lobby ID and version range, but keep account data and credentials out of labels; high-cardinality identity fields belong in controlled logs or traces.

Test the contract with disorder on purpose. Deliver version 12 before 11. Duplicate a command. Expire the retained range while a client is offline. Close the lobby between recovery and token issuance. Each case should end in one of two outcomes: a contiguous state at a known version, or an explicit rejection. Silent best-effort merging is not a third outcome.

## Deployment and Cost Boundaries

Retention is the main sizing lever. A longer change window reduces snapshot fallbacks but consumes storage and makes schema support last longer. A shorter window shifts work toward snapshot reads. Measure actual disconnect duration and change volume, then choose a window from those distributions; a made-up universal duration will be wrong for somebody.

Deploy additive change schemas first. Readers should tolerate fields they do not use, while writers must preserve the version contract throughout a rolling release. If an old reader cannot interpret a new change kind, return a snapshot in a schema it supports or require a client update explicitly. Do not let it guess.

The same restraint applies to capacity. Bound recovery page size, cap buffered live messages, and shed excess reconnect work before it starves ordinary room commands. Load tests should include synchronized reconnects plus active-room traffic, because testing either stream alone misses contention in the shared store.

## Limits

This design optimizes convergence on current lobby truth. It does not preserve every transient UI animation, replace a financial audit ledger, or define WebRTC signaling. Those are separate contracts with different retention and correctness requirements.

The decision rule is compact: choose snapshots when current truth is enough; add bounded changes when reconnect efficiency matters; adopt a durable log only when the full sequence is itself required. Keep scoped video authorization server-owned in all three.

## Sources

References:

- W3C, “WebRTC: Real-Time Communication in Browsers”: https://www.w3.org/TR/webrtc/
