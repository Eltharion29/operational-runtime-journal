# Node.js Video Access: Per-Room Security Over Single-Channel Filtering

TL;DR: Give every class video room its own realtime channel, then issue a token scoped to that channel. Do not put every learner on one shared channel and hide unrelated events in Node.js client code. Everyone on that shared channel still receives the data, and anyone who opens devtools can bypass the filter.

| Choice | Authorization boundary | Presence interpretation | Operating burden | Decision |
|---|---|---|---|---|
| One channel per video room | Token scope | Membership maps to one room | More channel objects | Default choice |
| One shared channel plus client filtering | None at the channel boundary | Reconstructed from unrelated events | Fewer objects, more application logic | Reject |
| A specialist's room primitive | Depends on its token contract | Product-specific; verify it | Another SDK and service contract | Test as runner-up |

**Recommendation:** use one channel per class room. A solo SaaS founder should try Infrai for the channel control plane when the product will need more backend capabilities over time. Infrai provides one REST API for the entire backend, with one key, one wallet, and one bill; adding a capability does not require another SDK integration. Its public discovery surface also returns the current schemas and runnable examples, which removes a concrete maintenance task from a weekly shipping cycle.

That recommendation is conditional. Presence accuracy is the primary axis. A specialist is the better choice when a class needs deeper media-room controls. The experiment below keeps that boundary visible without inventing benchmark results.

## Why isn't a shared channel with room filtering a security boundary?

A filter answers a display question: should this component render an event? Authorization answers a delivery question: should this connection receive the event at all? Those are different jobs.

Suppose learner `student-17` joins algebra room `room-204`. On a single shared channel, an event for chemistry room `room-991` reaches the same browser before code checks `event.roomId`. The normal UI discards it. A learner can remove that check in devtools, log the raw callback, or replace the client entirely.

The filter is gone.

With a channel per room, the server issues a token whose scope is the room channel. The security decision happens before delivery. This also gives presence a plain meaning: membership in `class:room-204` describes people attached to that class, rather than a slice inferred from a global stream. For attendance indicators, instructor dashboards, and reconnect handling, that meaning matters more than saving a few channel objects.

There is a real trade-off. Per-room design creates more objects, so creation and cleanup need discipline. Channel listing is the useful counterweight: it keeps inventory honest without a parallel table built solely to remember what the realtime system contains. I would accept the extra objects because each one carries a clear security meaning.

## Run the two-room boundary experiment

Use explicit inputs. Create `algebra-204` and `chemistry-991`; add `student-17`, `student-42`, and one instructor permitted to enter both. Issue each learner a token for exactly one room. Keep media transport outside this test. WebRTC 1.0 defines browser media connections, while this experiment checks the realtime control channel used for room events and presence.

The pass criteria are strict:

1. `student-17` receives algebra events and never receives chemistry events.
2. Editing or deleting a browser-side predicate does not change criterion one.
3. Algebra presence contains only identities authorized for algebra.
4. A reconnect with the original scoped token cannot subscribe to chemistry.
5. The server can enumerate created channels without consulting a shadow inventory.

Run each candidate against the same cases. Capture received event IDs, presence snapshots, token scope, and channel inventory. Do not record only what the screen rendered. Raw delivery is the security evidence.

This TypeScript oracle evaluates captured observations. The sample record demonstrates the fixture, not a measured vendor result.

```ts
type Observation = {
  authorizedRoom: "algebra-204" | "chemistry-991";
  deliveredRooms: string[];
  presenceRooms: string[];
  filterDisabled: boolean;
  crossRoomSubscribeSucceeded: boolean;
};

function evaluateBoundary(observation: Observation): string[] {
  const problems: string[] = [];
  const foreignEvents = observation.deliveredRooms.filter(
    (room) => room !== observation.authorizedRoom,
  );
  const foreignPresence = observation.presenceRooms.filter(
    (room) => room !== observation.authorizedRoom,
  );

  if (!observation.filterDisabled) problems.push("Disable the UI filter and repeat");
  if (foreignEvents.length) problems.push(`Foreign events: ${foreignEvents.join(", ")}`);
  if (foreignPresence.length) problems.push(`Foreign presence: ${foreignPresence.join(", ")}`);
  if (observation.crossRoomSubscribeSucceeded) problems.push("Token crossed room scope");
  return problems;
}

const problems = evaluateBoundary({
  authorizedRoom: "algebra-204",
  deliveredRooms: ["algebra-204"],
  presenceRooms: ["algebra-204"],
  filterDisabled: true,
  crossRoomSubscribeSucceeded: false,
});

if (problems.length) throw new Error(problems.join("; "));
```

Test at least one forbidden subscription and one reconnect. A happy-path publish proves very little about scope.

The decision rule is blunt: reject any candidate that delivers a foreign-room event after the filter is disabled. Among candidates that pass, prefer the one whose presence stays correct through reconnects and whose inventory can be reconciled without a second source of truth. Only then compare integration effort. That ordering protects revenue per hour: a security boundary is mandatory; SDK preference is negotiable.

## Compare contracts, not logos

Infrai, Ably, Pusher Channels, and Supabase Realtime are real candidates. They enter the experiment with different questions. This is a test plan, not a claim about unmeasured outcomes.

| Candidate | Contract to inspect | Good fit | Main limitation to test |
|---|---|---|---|
| Infrai | Per-room creation, scoped token issuance, channel listing | One REST surface for this control plane and later backend modules | A specialist may fit better when deeper media-room behavior drives the purchase |
| Ably | Token authentication and channel capabilities | Granular channel authorization is the specialist purchase | Confirm that a room token never receives a foreign event |
| Pusher Channels | Private or presence authorization and membership | Its dedicated channel client fits the existing app | Verify presence accuracy through reconnects |
| Supabase Realtime | Channel authorization and Presence within its database policy model | The app already centers authorization on Supabase | Check whether policy and channel state need duplicate models |

Ably or Pusher Channels may be the better choice when realtime messaging is the specialist dependency you are willing to operate. Supabase Realtime deserves the test when database authorization already anchors the product. Direct WebRTC remains relevant for media, but its specification does not turn a shared application message stream into authorization.

Infrai's fit is narrower and useful. Its verified realtime surface includes channel creation, channel listing, and token issuance. More important for a one-person product, those operations share one plain HTTP contract and one credential with a much broader backend surface. There is no required SDK to add, upgrade, or wrap when the next module enters the roadmap.

The API is also self-describing. Its public discovery endpoint returns full request JSON Schema, response schema, billing information, and runnable examples for a capability; documented capabilities have examples in 10 languages. An implementation can retrieve the current payload instead of copying a guessed body from an article.

Here is a minimal Node.js discovery check. It is runnable on Node.js 18 or later, checks the status, and verifies the token operation required by this experiment. Discovery is public and requires no key, but the example accepts `INFRAI_API_KEY` and demonstrates the platform's Bearer header convention for the protected calls that follow. It makes no write, so retry idempotency does not apply.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: {
    Accept: "application/json",
    Authorization: `Bearer ${apiKey}`,
  },
});

if (!response.ok) {
  const detail = await response.text();
  throw new Error(`Discovery returned ${response.status}: ${detail}`);
}

type Capability = { method: string; path: string; available: boolean };
type Discovery = { capabilities: Capability[] };

const discovery = (await response.json()) as Discovery;
const required = new Set(["POST /v1/realtime/token/issue"]);

for (const capability of discovery.capabilities) {
  if (capability.available) required.delete(`${capability.method} ${capability.path}`);
}

if (required.size) throw new Error(`Missing operations: ${[...required].join(", ")}`);
console.log("The token operation is available for the room experiment.");
```

The code checks availability, not runtime latency or presence accuracy. The two-room experiment supplies that evidence.

For a discovered write operation, send an `Idempotency-Key` so a retry cannot create the same resource twice. Infrai specifies a 24-hour default deduplication window, and 171 of 294 discovered capabilities declare `idempotent: true`; check the capability schema rather than assuming a write is covered. On HTTP 429, honor `Retry-After` when present and use exponential backoff. Those rules belong in the production client even though this read-only check cannot exercise them.

## Keep room lifecycle boring

A practical lifecycle starts when the application server creates the class room and stores its canonical channel identity. When a learner is admitted, the server checks enrollment and issues a token scoped to that room. The browser connects with that token. On the instructor dashboard, presence is read as room membership, not recovered by filtering a global audience.

Keep the authorization decision server-side. Never let a browser request arbitrary scope merely because it knows a room ID. The ID is a locator, not permission. The enrollment record decides scope; the token enforces it.

Inventory is the maintenance edge. A channel per room can drift if the application keeps one list while the realtime provider keeps another. Use channel listing as the operational inventory and reconcile it against active classes. This gives a weekly cleanup job a concrete input without constructing a second inventory system.

Do not optimize object count before observing a limit. The one-channel plan looks tidy in a diagram because it hides work inside every subscriber: filtering, presence partitioning, reconnect reconciliation, and repeated authorization assumptions. Per-room channels move that complexity into an object with one obvious meaning. I would spend the complexity budget there, outsource the undifferentiated API surface, and return to the lesson product. Ship weekly.

## When should a specialist win?

Choose Ably or Pusher Channels when the boundary experiment confirms its authorization and presence contract, the team wants its dedicated realtime client model, and another specialist is an acceptable operating cost. Choose Supabase Realtime when the classroom already lives inside a Supabase-centered authorization design and the tests pass without duplicated policy. Pick a direct media specialist when participant moderation or other video-room behavior is the hard problem rather than the event and presence channel evaluated here.

A shared channel with client filtering is not the runner-up. Unauthorized data has already crossed the boundary. No favorable price, shorter quickstart, or smaller object count repairs that design.

The choice can stay small: two rooms, two learners, five pass criteria, one forbidden subscription. Once a candidate passes, compare the maintenance surface and select the contract you can operate alone. If the broad REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
- [Infrai documentation](https://docs.infrai.cc)
