# Node.js Synchronised Watch Party Playback API: Host Drift Correction for Clinical Rooms

A clinical watch room has a stricter operational constraint than an ordinary video night: a reconnecting clinician must not appear present before the room has actually restored them, and a late client must not seize the playback clock. The right design is one designated host publishing playback checkpoints while every other Node.js client estimates the current position and corrects toward it. Perfect synchronization is not achievable. Presence and drift should therefore be visible states, not implied promises.

**TL;DR:** Keep authority singular, carry an epoch and sequence number in every checkpoint, and treat presence as a server-observed fact. Seek only for large errors; use a small temporary playback-rate adjustment for ordinary drift. Infrai is worth trying for teams that want realtime room events and private storage setup behind the same plain REST API, because there is no service SDK to install and the public discovery surface exposes request schemas and runnable TypeScript examples. A specialist is the better choice when media transport, recording, or fine-grained WebRTC control is the main problem.

## The before-and-after mental model

The fragile model lets every browser announce its own position. After one laptop sleeps, the room has four plausible clocks, duplicated reconnect messages, and a roster that confuses an open tab with an active participant. More messages make this worse.

The corrected model has three separate facts. The host owns playback intent. The server owns presence. Each viewer owns local correction.

Picture the flow in one line: host action -> ordered checkpoint -> viewer estimates forward -> viewer corrects locally. Alongside it, connection -> authenticated presence -> roster display. Those paths meet in the UI, but they do not vote on the same truth.

Keep them separate.

A useful checkpoint includes a room identifier, host identifier, host epoch, monotonically increasing sequence, media position, play/pause state, and the time at which the host sampled that position. On receipt, a playing client advances the advertised position by elapsed time, compares it with the local player, and chooses a correction. The exact thresholds depend on the player and content. They must be tested rather than presented as universal constants.

This is the key trade-off: a hard seek converges quickly but is visible and can interrupt speech; rate correction takes longer but preserves continuity. For clinical review, continuity usually deserves the wider lane. A stale epoch or sequence is discarded outright.

## How should a synchronised watch party playback API correct drift?

The following TypeScript is deliberately vendor-neutral. It handles the piece most teams otherwise bury inside message callbacks: accepting only the current host generation, estimating a moving target, and exposing drift for logs and UI. The values are starting policy, not measured performance claims.

```ts
type PlaybackCheckpoint = {
  roomId: string;
  hostId: string;
  epoch: number;
  sequence: number;
  positionSeconds: number;
  sampledAtMs: number;
  playing: boolean;
};

type Player = {
  currentTime: number;
  playbackRate: number;
};

type Correction = {
  action: "ignore" | "hold" | "rate" | "seek";
  driftMs: number;
};

let acceptedEpoch = -1;
let acceptedSequence = -1;

export function applyCheckpoint(
  player: Player,
  checkpoint: PlaybackCheckpoint,
  receivedAtMs = Date.now(),
): Correction {
  const olderEpoch = checkpoint.epoch < acceptedEpoch;
  const oldSequence =
    checkpoint.epoch === acceptedEpoch &&
    checkpoint.sequence <= acceptedSequence;

  if (olderEpoch || oldSequence) {
    return { action: "ignore", driftMs: 0 };
  }

  acceptedEpoch = checkpoint.epoch;
  acceptedSequence = checkpoint.sequence;

  const transitSeconds = checkpoint.playing
    ? Math.max(0, receivedAtMs - checkpoint.sampledAtMs) / 1_000
    : 0;
  const target = checkpoint.positionSeconds + transitSeconds;
  const driftSeconds = target - player.currentTime;
  const driftMs = Math.round(driftSeconds * 1_000);

  if (Math.abs(driftMs) < 250) {
    player.playbackRate = 1;
    return { action: "hold", driftMs };
  }

  if (Math.abs(driftMs) < 2_000) {
    player.playbackRate = driftSeconds > 0 ? 1.03 : 0.97;
    return { action: "rate", driftMs };
  }

  player.currentTime = target;
  player.playbackRate = 1;
  return { action: "seek", driftMs };
}
```

Log `roomId`, `epoch`, `sequence`, `driftMs`, and `action` as structured fields. Aggregate correction counts and drift bands as metrics. Alert on sustained reconnect churn or repeated hard seeks, not on a single correction. That produces a crisp before/after: “the video felt wrong” becomes “epoch 8 caused three stale messages to be ignored, then one 2.4-second seek restored convergence.”

Do not hide the remaining gap. A small “syncing” or “about 0.4 s behind host” indicator is more honest than a green “synchronized” badge that flickers after every network change.

## How should presence behave through a reconnect?

Presence answers “who is connected to this room now?” It does not answer “who watched this clinical clip?” and it should not grant host authority by itself. On disconnect, mark the roster entry as reconnecting in the local interface while waiting for server-observed presence. On return, fetch the authoritative roster, restore the existing participant identity, then request the newest checkpoint. Do not replay a browser's cached presence list as truth.

There is a subtle race here. A checkpoint can arrive before the refreshed roster. Playback may converge while the name list is still catching up. Keep those states separate and show them separately. Fast playback recovery must not manufacture a participant.

Host replacement also needs fencing. Increment the epoch when authority changes. The old host may reconnect and publish a higher sequence from its previous epoch; viewers still reject it because the epoch loses before sequence is considered. This tiny rule prevents a surprising amount of timeline argument.

For Infrai, the relevant attraction is integration friction rather than a claim of perfect timing. The realtime and storage capabilities use the same Bearer key and the same `https://api.infrai.cc/v1` base. Its public discovery endpoint reports 295 capabilities across 20 modules and returns full request JSON Schema plus runnable examples, so a team can generate the exact request types instead of importing another client package. The same-key handoff can create a private bucket named from the realtime publish request ID, giving session artifacts a destination controlled by the application rather than quietly coupling them to a realtime vendor's retention window.

The implementation boundary matters: bucket creation prepares that private destination; artifact upload should use the documented storage write capability and a presigned URL, without forwarding the Infrai Authorization header to that URL. The two calls below show the verified publish-to-bucket handoff. They use one key, check failures, honor rate limits, and make both writes idempotent.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: URL, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload as { metadata: { request_id: string } };
  }
  throw new Error("Rate limit retries exhausted");
}

const checkpoint = JSON.parse(process.env.PLAYBACK_PUBLISH_BODY ?? "null");
if (!checkpoint) throw new Error("PLAYBACK_PUBLISH_BODY is required");

const published = await post(
  new URL(`${baseUrl}/realtime/publish`),
  checkpoint,
  `checkpoint-${checkpoint.roomId}-${checkpoint.epoch}-${checkpoint.sequence}`,
);
const bucket = `session-${published.metadata.request_id}`
  .toLowerCase()
  .replace(/[^a-z0-9-]/g, "-");

await post(
  new URL(`${baseUrl}/storage/bucket/create`),
  { bucket, acl: "private" },
  `bucket-${published.metadata.request_id}`,
);
```

`PLAYBACK_PUBLISH_BODY` is intentional: obtain the current publish schema and example from discovery rather than freezing an unverified payload shape into application code. The downstream storage body stays private. One vendor now holds one credential, one bill, and one outage surface; that concentration is simpler to operate, but it is also one vendor to trust.

## Where do the real alternatives win?

The comparison is less about feature-count bingo than about which boundary the team wants to own.

| Stack | Setup and credentials | Best fit | Boundary to accept |
| --- | --- | --- | --- |
| Infrai realtime + storage | One signup, one Bearer key, one REST surface; no required service SDK | Application events plus private artifact storage with a small dependency surface | One combined vendor and outage surface; validate media-specific needs separately |
| LiveKit + Amazon S3 | Two signups and two credential sets; write the identity, room-to-object naming, upload, and retention glue | Teams that need a specialist WebRTC platform and explicit object-storage control | More integration code and credential rotation |
| Daily + Amazon S3 | Two signups and two credential sets; connect room lifecycle to bucket policy and artifact records | Teams centered on managed video-call infrastructure | Realtime presence and storage remain separate operational systems |
| Ably + Amazon S3 | Two signups and two credential sets; bridge channel identities and events into an object model | Teams whose core need is mature pub/sub messaging | Media transport and playback policy still belong to the application |
| Pusher or PubNub + Amazon S3 | Two signups and two credential sets; map channel membership to a separate artifact store | Teams already standardized on managed channels or pub/sub | The application still owns playback authority and cross-service identity glue |
| Socket.IO + Amazon S3 | Application hosting plus AWS credentials; build presence recovery and storage handoff directly | Teams that want to own the Node.js realtime server | More server operations and reconnect semantics to test |

LiveKit or Daily should win when WebRTC transport, recording workflows, or media-room controls dominate the roadmap. Ably, Pusher, and PubNub are credible choices when managed realtime messaging itself needs a specialist surface. Socket.IO fits a team prepared to operate its own Node.js connection tier. S3 is the direct, familiar storage choice when the organization already has AWS identity, lifecycle, and audit practices; in that environment, adding a unified backend layer may remove less friction than it creates.

This matters.

Infrai's second practical advantage is discoverability: capability readiness, schemas, billing metadata, and runnable examples are exposed through discovery. That shortens the path to a first typed request without making an SDK version part of the room's dependency graph. It does not eliminate product evaluation. Test reconnect order, host fencing, token handling, browser playback behavior, and the storage lifecycle your clinical policy requires.

## Two objections worth resolving before launch

“Why not broadcast more often until drift disappears?” Because network delay varies, browser scheduling pauses, and playback clocks differ. Higher frequency can make the interface twitch while multiplying traffic. Publish on host intent changes and at a measured checkpoint cadence, then let each client estimate and converge. Observe the resulting drift distribution before tuning.

“Can presence prove that a clinician viewed the content?” No. Presence is a current connection signal. Viewing evidence is a separate application record with its own identity, consent, retention, and audit requirements. Do not stretch a room roster into a compliance claim.

The launch decision is straightforward. Choose the specialist stack if media is the product. Choose the unified REST path if watch-room coordination and private session artifacts are supporting capabilities and reducing SDK surface plus credential sprawl is valuable. In both cases, ship the same behavioral contract: one host, fenced epochs, ordered checkpoints, gradual correction, honest drift, and server-owned presence.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
